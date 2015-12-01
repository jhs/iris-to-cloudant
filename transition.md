# How To Migrate from Iris Couch to IBM Cloudant

Both IBM Cloudant and Iris Couch are compatible with the open source Apache CouchDB database; and so they are compatible with each other. That's great news! It means that you can easily migrate from Iris Couch to Cloudant with a minimum of troubles, using your web browser.

This document will walk through the process of migrating your data from Iris Couch to Cloudant. Here is the migration plan:

1. Create a free IBM Cloudant account
1. Use the replication command to move your data from Iris Couch to Cloudant
1. Configure virtual host settings (if you have any)
1. Update your application
1. Understand the minor differences between Cloudant and Iris Couch

Both Cloudant and Iris Couch are compatible (and based on) Apache CouchDB; and CouchDB excels at replication. Thus Iris Couch is a very convenient migration platform--both to it and from it. Most people can completely migrate in an hour or so.

Indeed, Iris Couch has a very helpful blog post, [How-To: Bail out on Iris Couch][iris-bail]. That walkthough is great for moving to Apache Couchdb; however, this document will focus specifically on Cloudant, a quicker and simpler task. So let's begin!

## Preparation

In this document, we will use the example, *example.cloudant.com* and *example.iriscouch.com*; but of course your account will be your own, with your own user name prefixed before the `.cloudant.com` and `.iriscouch.com` domain. In all cases, substitute your own account's URL instead of *example.cloudant.com* and *example.iriscouch.com*.

Before you begin this migration, you will need to know:

1. Your Iris Couch URL, for example, *example.iriscouch.com*
1. Your Iris Couch admin username and password. To confirm these:
  1. Go to Futon: *https://example.iriscouch.com:6984/_utils/*
  1. Click the *Login* link in the lower-right ![Futon Login link example](img/futon-login.png)
  1. Input your admin username and password in the prompt ![Futon login prompt](img/futon-prompt.png)

https://jhs.iriscouch.com:6984/_utils/

## Create a Free IBM Cloudant Account

IBM Cloudant is free! [Sign up for a free Cloudant account][sign-up]. Once you have signed up, [sign in][sign-in].

## Set a temporary 
[TODO]: dns,cors
[END]: ------------------------------------------------------------------------
[iris-bail]: http://www.iriscouch.com/blog/2011/05/how-to-bail-out-on-iris-couch
[sign-in]: https://cloudant.com/sign-in/
[sign-up]: https://cloudant.com/sign-up/

Thanks to its CouchDB foundation, Iris Couch is an easy 

This is a basic disaster-recovery (DR) scenario. We want to perform one simple task to completely pull the plug, jumping to a different CouchDB system.

Thus these are the main objectives:

* Have a duplicate of the **Iris couch**, called **B-Couch**.
* B-Couch syncs from Iris couch automatically or regularly.
* Be able to activate B-Couch with a single domain name change.

## Quick and Dirty Sidenote

If you have no backups whatsoever, the dead-simple *partial solution* is to make a full document dump: `/my_db/_changes?include_docs=true`

This is like an **incomplete mysqldump** because it does not fetch document attachments. On the other hand, it's a quick one-liner which you can run from cron, so what's stopping you?

    .no-highlight
    $ curl 'https://icuser:icsec@ex.iriscouch.com/db/_changes?include_docs=true'
    {"results":[
    {"seq":5,"id":"doc 1","doc":{"_id":"doc 1","_rev":"7-1f6e64...","awesome":true}, etc. },
    {"seq":8,"id":"other doc","doc":{"_id":"other doc","_rev":"3-8b92cd..."}, etc. },
    { ... etc ... },
    ],
    "last_seq":97}

**This is not a complete copy** of the data. Again: it will not include attachments.

However, if you have no backups in the first place, well, at least it's a full replica of the data. Bang for buck.

Next I'll explore a proper couch clone. Yeah. We all see what's coming. We're going to *replicate* CouchDB&mdash;no surprises there. Of course, "surprise" isn't typically part of a good DR plan.

## Preparing the Iris couch

First, you need a user account on the Iris couch. This account must have permission to read every database. In a pinch, that can be the system `_admin` account.

For this discussion, suppose the user name is *icuser* and the password is *icsec*, on the couch *ex.iriscouch.com*.

Confirm that *icuser* (or any of *icuser*'s roles) can access every database. The security object (in Futon's "Security" section) for each database should list *icuser* in the "users" array, or one of *icuser*'s roles in the "roles" array.

In other words:

    .no-highlight
    * For each database:
      * Check the _security object (the "Security" link in Futon)
      * Check the "Admins" section:
        * If Names contains icuser, ok.
        * If Roles contains any of icuser's roles, ok.
      * Check _security (the Security link in Futon) in the Readers section:
        * If Names and Roles are both empty, ok.
        * If Names contains icuser, ok.
        * If Roles contains any of icuser's roles, ok.
      * Otherwise, you must add icuser (or its role) somewhere in this mix.

You can confirm that everything works with `curl`.

    .no-highlight
    $ curl https://icuser:icsec@ex.iriscouch.com/_all_dbs
    ["stuff","not_stuff"]

    $ curl https://icuser:icsec@ex.iriscouch.com/stuff
    {"db_name":"stuff","doc_count":92,"doc_del_count":12,... // etc.

    $ curl https://icuser:icsec@ex.iriscouch.com/not_stuff
    {"db_name":"not_stuff","doc_count":8,"doc_del_count":0,... // etc.

## Preparing B-Couch

Obviously, you need **B-Couch** on-line. This can be pretty much any CouchDB server you can find or run yourself. In fact, B-Couch can be on your laptop, or a spare computer in the office. If you haven't got *any* clone of your couch, by all means, start with CouchDBX, homebrew, or (if you're sophisticated) the CouchDB that comes with your OS.

If B-Couch is private, you won't have the DNS switchover we discuss later. But again: that's better than nothing.

Iris Couch runs the GeoCouch plugin. An easy way to build CouchDB with GeoCouch is with [Build CouchDB][build]. (Well, you might say "an easier way.") **GeoCouch is not necessary to have a complete backup**; however, if you want to use B-Couch in production with geo queries, you'll need it.

[build]: https://github.com/iriscouch/build-couchdb

As before, you need a Couch user with write access to the databases. You could use the `_admin` account. Nobody'd blame you. But a less-privileged account is better.

A good place to start is replicating the user database straightaway.

    .no-highlight
    $ curl -X POST http://admin:secret@couch.b-couch.com/_replicator/      \
      -H 'Content-Type: application/json'                            \
      -d '{ "source":"http://icuser:icsec@ex.iriscouch.com/_users"}  \
          , "target": "_users"}'

Replicating the `_users` database will add identical account names and roles to B-Couch. If you don't want peopla *using* B-Couch, then set the `_security` objects differently on B-Couch. See? Locked down. That's why security objects don't replicate.

Confirm that B-Couch responds to queries and that everything looks good.

    .no-highlight
    $ curl https://icuser:icsec@couch.b-couch.com/
    {"couchdb":"Welcome","version":"1.1.0"}

    $ curl https://icuser:icsec@couch.b-couch.com/_session
    { "ok": true
    , "userCtx": { "name": "icuser", "roles": ["some_role"] }
    , "info": { "authentication_db": "_users"
              , "authentication_handlers": [ "oauth", "cookie", "default" ]
              , "authenticated": "default"
              }
    }

## Replicate

You did it once already and it wasn't so bad. Do that for every database you need.

    .no-highlight
    for db in admin_db _users some_other_db; do
      curl -X POST http://admin:secret@couch.b-couch.com/_replicator/    \
           -H 'Content-Type: application/json'                           \
           -d '{ "source":"http://icuser:icsec@ex.iriscouch.com/_users"} \
               , "target": "_users"}'
    done

Here our paths diverge a bit. This is *your* back-up, DR, cold-spare, hot-spare, or whatever. The point is, you've got your own couch that is a clone of your Iris couch. Back it up to tape. Put it in Dropbox. Whatever.

The command above **does not use continuous replication**.  Instead, it starts one-off replications, effective immediately. That's easy to understand, easy to keep an eye on, and easy to back up correctly.

You might prefer a *continuous* replication, maintaining a real-time copy of the Iris couch. Any time, rain or shine, B-Couch will be identical to the Iris couch. If that sounds like something you want, add one more field to your replication requests:

    .javascript
    { "source": "http://icuser:icsec@ex.iriscouch.com/_users"}
    , "target": "_users"
    , "continuous": true
    }

## The hostname switch

Finally, the big red button!

We want one simple place where we can change one configuration or setting, and B-Couch takes over in production.

The answer is to use your own DNS domain. For example, suppose your domain is **example.com**. Build your applications and services to use CouchDB from **couch.example.com**.

    .no-highlight
    ; Example BIND 9 setting. This points a domain name I
    ; control (couch.example.com) to whichever couch I
    ; prefer (currently Iris Couch).
    couch           CNAME   ex.iriscouch.com.

Consider the time-to-live (TTL) for your domain. When you change the CNAME, users will still connect to the Iris couch for up to *TTL* seconds.

Confirm that your CNAME looks good and works for queries.

    .no-highlight
    $ dig couch.example.com A

    ;; QUESTION SECTION:
    ;couch.example.com.             IN      A

    ;; ANSWER SECTION:
    couch.example.com.      712     IN      CNAME   ex.iriscouch.com.
    ex.iriscouch.com.       712     IN      A       184.73.200.44

    $ curl https://icuser:icsec@couch.example.com/
    {"couchdb":"Welcome","version":"1.1.0"}

**You're done!**

Now, if you ever need to change which couch you use for production, just update the CNAME record in your DNS domain. No software changes, no downtime, no lock-in.

    .no-highlight
    ; Modified BIND 9 setting. This points a domain name I
    ; control (couch.example.com) to whichever couch I
    ; prefer (now B-Couch).
    couch           CNAME   couch.b-couch.com.

When that pushes and propogates, the client-side (that is, your web app or iPhone app) doesn't change.

    .no-highlight
    $ dig couch.example.com A

    ;; QUESTION SECTION:
    ;couch.example.com.             IN      A

    ;; ANSWER SECTION:
    couch.example.com.      811     IN      CNAME   couch.b-couch.com.
    couch.b-couch.com.      811     IN      A       1.2.3.4

Note that the couch clients require no change at all.

    .no-highlight
    $ curl http://couch.example.com/
    {"couchdb":"Welcome","version":"1.1.0"}