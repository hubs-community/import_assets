import_assets
=============

A python script and associated SQL and bash commands to migrate Hubs cloud data.

A rough description of how to decrypt the assets stored in Hubs is also described. No script is available (yet) which benefits from this, but there is not reason it would not be possible.

1. [Description of the script](#description-of-the-script)
2. [About assets encryption](#about-assets-encryption)


Description of the script
-------------------------

Described below is one potential way to migrate data from an existing Hubs cloud stack to a new database and filesystem.

Apologies in advance for the giant wall of text, this was originally a series of posts to the Hubs community discord.

Okay, so my goal was to copy over everything useful out of my Hubs database, and grab all the images, scenes, models, etc. that had been uploaded, and migrate them over to my CE stack with account_id links intact.

First, the database. For reasons that are stupid (different versions of pg_dump and pgsql) I wasn't able to use pg_dump, so instead I went in and used \COPY to export a table at a time, but do whatever works best for you. It turned out that for my purposes (given the limited set of Hubs features we use at MeTabi) I only actually needed six tables out of the whole db:  accounts, owned_files, assets, scenes, projects, and Hubs.

So, with that data exported and imported, I had one problem: the account_ids for my new user accounts didn't match my old user accounts referenced in all the tables I just copied. As I write this I am mildly embarrassed to note that I could have just changed the new account ids to match the old ones in the accounts table, but instead I wrote a few SQL statements to change them in all the other tables. 🙂 (I only have two active users, so it just took a couple of minutes). But anyway, enough said about that... my tables now matched my current users and we're good to go.

But now comes the complicated part - the actual files.
Hubs does not keep all of the files you upload in a neat little directory where you can just go grab them. First of all, they are all encrypted, and they have all been named with UUIDs, so if you uploaded "myCoolImage.jpg", what you will see when you finally actually find it is something like "d4bef7d3-8f4b-4e2e-87a6-717622d147dc.blob" sitting next to "d4bef7d3-8f4b-4e2e-87a6-717622d147dc.meta.json".

But where are you going to find these files?? I had very little idea, but I knew that Alex's new Helm chart had made my reticulum and pgsql pods use a persistent EFS filesystem for storage. And I could see that there was a folder named "storage" at the root drive of my reticulum pod, as well as my Hubs cloud reticulum instance.

Here is where it gets tricky though, because while these files live at a URL that looks like "mydomain.com/files/<uuid>.jpg", when you open the storage folder, you see a bunch of folders, not named "files". Among them is one called "owned" and another called "expiring", and inside each of those you will find a pile of directories with two letter names, like "aa a1 a2 b2 e3 ...". Hmm... 🧐 
A bit more research quickly revealed that this is a common sharding technique to guarantee that we don't end up trying to access a directory with ten million files in it, because that is not performant. Instead, we have a system where we take the first two letters of our filename, and find or create a directory named with those two letters. Then, to take it a step further we add one more layer to this system by taking the next two letters and turning them into another directory. So, "d4bef7d3-8f4b-4e2e-87a6-717622d147dc.blob" ends up as "d4/be/d4bef7d3-8f4b-4e2e-87a6-717622d147dc.blob".

So, great, simple enough! However, the real job here is the encryption, and here is where my first plan fell apart. There was a hope that if I used the same Phoenix and/or Guardian keys with the new CE cluster as I was using with the old Hubs cloud stack, that it would "just work" and all my files would decrypt accurately down to the last bit. Sadly, this did not prove to be the case.

This problem most likely could have been solved in one way or another, but by this point my attention span was exhausted and I was tired of mucking around with encryption keys, so I elected to try a different approach. I knew that I could download all my assets, unencrypted, by using the pattern "https://<domain_name>/files/<uuid><extension>". The uuids of all my files can be found in the "owned_file_uuid" column of the owned_files table, and the content_type field gives me the extension, as follows: 

```python
    if type == 'application/json':
      ext = ".json"
    elif type == 'model/gltf-binary':
      ext = ".glb"
    elif type == "model/gltf":
      ext = ".glb"
    elif type == "image/png":
      ext = ".png"
    elif type == "image/jpeg":
      ext = ".jpg"
    elif type == "application/octet-stream":
      ext = ""
```
      
(Note that you need to put the "." into the extension itself, because the octet-stream object has none.)
The second part of my new plan was to call the Reticulum API directly to upload these files, using my new cluster's local encryption scheme. I did all of this from Cloudshell on AWS, so that the downloads and uploads would all be bouncing around on the same AWS servers and I would not have to download the content across the internet to my house and then send it back. It is important to note, though, that I deleted each file after I was done using it, before downloading the next one! This is because Cloudshell only gives a user one gigabyte of local storage to work with, and I am transferring 12 gigs total here.

In order to access Reticulum, however, we need to successfully shake its hand. Our TLS handshake will fail if we do not have a key, so where are we going to get that? It turns out what it needs is the token that we get when we sign into Hubs. You can see this token in your browser local storage whenever you are signed in to your Hubs instance, either on the homepage or in a room. All you have to do is open your Web Tools, drag the panel out real wide until you can find the Storage tab, go into Local Storage, and open the section under your Hubs domain. There should be an entry called `__Hubs_store` and inside that you should find "credentials", open that up and find "token". That's what you need.

To make things easier, I put that into a file, and then load that file into the python script I'm writing. The details of the API access look like this, using the python "requests" library:

```python
  api_endpoint = "https://<Hubs_domain>/api/v1/media"
  auth_header = { 
    "Authorization" : "bearer " + token
  }
  destination = "files/" + uuid + ext
  shortname = uuid + ext
  formData = {"media": (shortname, open(destination,mode='rb'),type)}
  response = requests.post(
     url=api_endpoint,
     headers=auth_header,
     verify=False,
     files=formData
  )
```
When this response comes back, it will have some json in it that looks like this:

```json
  {"file_id":"579744ca-9cda-4358-9c79-5a8d49110a5c","meta":{"access_token":"32ee5874c4a8d6ea26c71bdb0353fb03","expected_content_type":"image/jpeg","promotion_token":null},"origin":"https://metabi-poc-hub.com/files/579744ca-9cda-4358-9c79-5a8d49110a5c.jpg"}
```

So, great! I have now made a system with which to upload all of my assets to the new stack. W00t! Are we done then?

Ha! You knew we weren't done yet. The next problem: When we upload a file using the Reticulum API, it not only encodes it, it also gives it a new UUID name. This is seen in the "file_id" field, as well as the "origin" URL. For purposes of our migration, this is a most unfortunate behavior, and an improvement I would love to make here is to find a way to make that optional. Because now I have 3000 files that all have new names and are not going to be linked properly.
So, now comes the fun part! First, the database is pretty easy. While I am downloading and uploading my files, I write a file called update.sql, into which I add the following, for each uploaded file:  "UPDATE owned_files SET owned_file_uuid=<new_uuid>, key=<new_token> WHERE owned_file_uuid=<old_uuid>;".

Then, since I am running this script in my Cloudshell instance where I have direct access to kubtctl and my cluster, I can simply run the following from python via subprocess:

```python
  subprocess.call("kubectl cp update.sql moz-pgsql-587fb8ccb8-8rtkj:/update.sql",shell=True)
  
  subprocess.call("kubectl exec -it moz-pgsql-587fb8ccb8-8rtkj -- psql  -U postgres -h localhost retdb < update.sql",shell=True)
```

Other people might write that into a bash script and call that instead, but either way, we're getting the job done.
So... NOW are we done? HA! Sadly not. There are two more things to do. First: all of the above was fine for all of my simple assets, like images and models, but a large part of my assets are json scene files. These json files have direct URLs pointing at my old Hubs stack domain, and of course the old UUIDs for all of the files. So, when I get to downloading and uploading these json files, I have to do some searching and replacing. The domain is simple, but the uuids get tricky.

To deal with this, I added another text file to my system: as I download each file, in addition to writing the SQL mentioned above, I also write a simple map file called uuid_map.txt, with nothing but comma separated lines like "<old_uuid>,<new_uuid>". Then, I made sure to separate my import CSV values by type, so that I could download/upload all of my simple assets before starting on any of the json files. Then I start my script for the json files, I load that entire map file into a python dictionary. Then, every time I see a domain to switch out, I also count characters back until I find the beginning of the uuid. (This isn't as hard as it sounds, since it is always the same distance.) Once I know the old uuid, I just reference my dictionary to get the new one, and do my search and replace.
Once that is done, I can upload the modified text and it gets encrypted and sent on up just like everything else, and life is good.

But... IS IT? It turns out that even with all of this being done, we have one more problem. That is the fact that when you upload files through reticulum the way I am doing it, you are basically simulating what happens when you drag a file into Hubs, without pinning it or uploading it to spoke. Namely, it is a temporary file. This means that instead of living in the "owned" folder under /storage, it lives in the "expiring" folder. This means two things: A) you cannot access it without including a "token" argument on the URL, which is the encryption key that was returned by reticulum after you uploaded it, and B) it will not last forever - I don't know the frequency, but the term "expiring" definitely leads us to believe it will probably be automatically deleted sooner or later.
However, trial and error taught me that if I simply move the files from "expiring" to "owned" then my problem is solved, as long as there is a record in the owned_files table that points to this uuid and has a valid key stored. To make copying the files more efficient, I kept a list of all the top level two-character directory names I accessed in each execution of the script, and then after I am all done with everything else, I go copy each of those whole directories over, using subprocess("kubectl cp ...") again.

And lastly, after all that, I go ahead and delete everything in the expiring directory, so I don't have an extra 12 gigs sitting around until the cron job decides to do housecleaning.

And THAT, gentle reader is finally the last step, we are done now! I hope this tale has been at least mildly interesting and informative for somebody. Like I said at the beginning, it is probably not the easiest way to go about this, but if you have trouble with the official way or you just want to know what's going on at a deeper level, here you go!

My final python script is included below, feel free to modify and make use of it if it can help you.

Happy Holidays!
Chris


About assets encryption
-----------------------

Once again, a rough introduction to how to dive into the assets encryption in Hubs. There might/will be approximations in this guide, so thank you for your understanding (and your insights!).

Note that this documentation was written from observations using a self-hosted Hubs instance, on Digital Ocean. Thnigs may vary when installed through the old AWS cloud stack, or using the new Hubs Community Edition. But overall the information should still apply, paths could be different.

Lastly, some bits of information will be repeated from the previous section. Maybe at some point it will be interesting to factorize all this.

### Data location

Most data is stored in `/storage` :

- permanent assets are stored in `/storage/owned`
- temporary assets added by users are stored in `/storage/expiring`
- Reticulum configuration is stored in `/storage/svc/reticulum/var/sys.config`

Assets are stored as `.blob` binary files, along a `json` file describing the content of the blob. The binary file can be a glTF, PNG, JPEG, etc. file ecrypted with a key very specific to this file. The key is stored in the database, in our case a PostgreSQL one. The key is hashed with a secret key which is specified in the Reticulum configuration file, look for the entry `secret_key_base` in the section `Elixir.RetWeb.Endpoint`.

From the same configuration file, you will also need the configuration information to the database. They can be found in the `Elixir.Ret.Repo` section.

Lastly, the encryption/decryption is handled by Reticulum through the use of the [Elixir programming language](https://elixir-lang.org/), which runs on top of Erlang. More specifically the code handling all this can be found [here]( https://github.com/mozilla/reticulum/blob/master/lib/ret/crypto.ex). It will be needed later on.

### Copying the desired file

Before anything, it is advised to copy the file you want to get back somewhere else, for example your own computer. It can be hard to devise which file is which, so the best is to guess from the file size, the description in the associated `json` file, the creation date, etc. You might have to do multiple tries until you find the right file.

### Getting the files encryption keys

Now for some step-by-step instructions. First we will extract the key from the database. Let's connect to it, with the credentials taken from the configuration file as well as the hostname / ip address of the database server, taken from the Digital Ocean project:

```bash
psql -h HOSTNAME -p 25060 -d polycosm_production -U doadmin
```

Then in `psql`:

```sql
\list
\connect polycosm_production
\dt
SELECT * FROM owned_files;
```

This will print information about all the files stored in `/storage/owned`, and in particular the encryption keys. Files can be associated using their UUID.

### Decrypting files

To decrypt the files, entirely manually, we will use directly the code from Reticulum. First you will have to install Elixir. For example, on Ubuntu and Fedora:

```bash
sudo apt install elixir
sudo dnf install elixir
```

Then run the Elixir interpreter:

```bash
iex
```

And copy/paste the whole [cryptographic implementation from Reticulum](https://github.com/mozilla/reticulum/blob/master/lib/ret/crypto.ex). This will make all its functions readily usable inside the interpreter. Next, still in the interpreter:

```erlang
# Check that we have the correct file path
File.state("/path/to/encrypted/file")

# Hash the key
key = <<"KEY_FROM_THE_SQL_TABLE">>
hashed_key = Ret.Crypto.hash(key, "SECRET_KEY_BASE_FROM_SYS.CONFIG")

# Decrypt the file
{ok, stream} = Ret.Crypto.decrypt_file_to_stream("/path/to/encrypted/file", hashed_key)

# Save the resulting stream to another file
outfile = File.stream!("/path/to/decrypted/file")
stream |> Enum.into(file)
```

And that's it, you now have your file decrypted! Take care of using the correct file extension, and verify that all is correct by importing it in an appropriate software.

### Final thoughts

It should be possible to automate all this, as well as decrypt the files using Python instead of Elixir. From my own understanding it seems that Reticulum places the encryption tag at the beginning of the encrypted data, and the Python Cryptography module expects it at the end.
