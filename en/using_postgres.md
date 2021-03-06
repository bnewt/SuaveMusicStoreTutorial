# Using Postgres

Now that we've moved to using Paket, let's replace the dependency on MS SQL Server with [Postgres](http://www.postgresql.org/).
I'd like here to say big thanks to Chris Holt ([@lefthandedgoat](https://twitter.com/lefthandedgoat)) for proposing the postgres substitute and implementing the Db module for Postgres, which I based on.

SQLProvider does support Postgres, but unfortunately we'll need to adjust all the tables, views and columns naming to be lowercase - that's because the mixture of SQLProvider + Npgsql + Postgres didn't quite work well with PascalCase names.

To start with, let's update the `SQLProvider` dependency to 0.0.11-alpha, and also add `Npgsql` package - .NET provider for PostgreSQL (`paket.dependencies`):

```bash
source https://www.nuget.org/api/v2

nuget SQLProvider 0.0.11-alpha
nuget Npgsql 2.2.7
nuget Suave 1.0.0
nuget Suave.Experimental 1.0.0
```

The `Npgsql` binaries will be necessary for the `SQLProvider` to work with Postgres database.
To include them in the target `bin` directory, add a proper entry to `paket.references` file:

```bash
SQLProvider
Npgsql
Suave
Suave.Experimental
```

Next, run `paket update`:

```bash
$ mono .paket/paket.exe update
```

> note that the command could probably update the `FSharp.Core` (transient) dependency to version 4 as well - this happened in my case

The SQL script used for creating the Database Schema in Postgres can be downloaded [from here](https://github.com/theimowski/SuaveMusicStore/blob/postgres/postgres/postgres_create.sql).

Before we can run the Suave Music Store on a *nix box we'll have to install and configure PostreSQL.
In case you haven't done that before (just as me prior to writing this article), [this](https://help.ubuntu.com/community/PostgreSQL) is a handy guide on how to do that.
When you have PostgreSQL configured, run the `postgres_create.sql` from within the `psql` interactive terminal:

```bash
$ \i .../postgres/postgres_create.sql
```

## Changes in F# sources

I've divided this subsection into three parts, each concerning a separate module (Db, View, App).
If you want to save time on applying the changes by yourself - you can just scroll down to end of each part where I included a link to module implementation after changes and copy relevant bits.

### Db module

First, we need to connect to the new database using proper SQLProvider configuration:

```fsharp
type Sql = 
    SqlDataProvider< 
        ConnectionString      = "Server=127.0.0.1;Database=suavemusicstore;User Id=suave;Password=1234;",
        DatabaseVendor        = Common.DatabaseProviderTypes.POSTGRESQL,
        CaseSensitivityChange = Common.CaseSensitivityChange.ORIGINAL >
```

Next, we have to amend aliases for our data types:

```fsharp
type DbContext = Sql.dataContext
type Album = DbContext.``public.albumsEntity``
type Artist = DbContext.``public.artistsEntity``
type Genre = DbContext.``public.genresEntity``
type AlbumDetails = DbContext.``public.albumdetailsEntity``
type User = DbContext.``public.usersEntity``
type Cart = DbContext.``public.cartsEntity``
type CartDetails = DbContext.``public.cartdetailsEntity``
type BestSeller = DbContext.``public.bestsellersEntity``
```

Then, adjust all references to the table sets. As an example `getGenres` should be changed from:

```fsharp
let getGenres (ctx : DbContext) : Genre list = 
    ctx.``[dbo].[Genres]`` |> Seq.toList
```

to:

```fsharp
let getGenres (ctx : DbContext) : Genre list = 
    ctx.Public.Genres |> Seq.toList
```

Next, change all column and table name references from PascalCase to Firstcapitalcase, e.g. from:

```fsharp
    on (album.GenreId = genre.GenreId)
```

to:

```fsharp
    on (album.Genreid = genre.Genreid)
```

And there's one more thing. Change the implementation of `createAlbum` to:

```fsharp
let createAlbum (artistId, genreId, price, title) (ctx : DbContext) =
    let a = ctx.Public.Albums.Create()
    a.Artistid <- artistId
    a.Genreid <- genreId
    a.Price <- price
    a.Title <- title
    ctx.SubmitUpdates()
```

The the previously used `Create` signature version taking all 4 values inline happened to be faulty for some reason.

[Db module after changes](https://github.com/theimowski/SuaveMusicStore/blob/postgres/Db.fs)

### View module

In view module, the only necessary changes regard column name case.
Similar to Db module, rename all column name references to PascalCase to Firstcapitalcase, e.g. :

```fsharp
(sprintf Path.Store.details album.AlbumId) 
```

goes to:

```fsharp
(sprintf Path.Store.details album.Albumid) 
``` 

[View module after changes](https://github.com/theimowski/SuaveMusicStore/blob/postgres/View.fs)

### App module

The same applies for App as for View module - replace each non-first uppercase letter with its lowercase equivalent.

[App module after changes](https://github.com/theimowski/SuaveMusicStore/blob/postgres/App.fs)

If you didn't encounter any issues, you should finally be able to compile and run the application.
We can prepare a `build.sh` script to include steps necessary for compiling the app:

```bash
#!/usr/bin/env bash

mono .paket/paket.bootstrapper.exe
mono .paket/paket.exe restore
xbuild
```

Run the `build.sh` script.
If there are any compile time errors, it's probably due to name conflicts - follow compile errors to troubleshoot.
Now fire up the application:

```bash
$ mono bin/Debug/SuaveMusicStore.exe
```

Navigate to 8083 port of your localhost to verify that the app is up and running.

## JavaScript bug

During preparation for this section, I discovered a nasty bug in this very complicated `script.js` sitting in the code base.
Here's a fix:

```js
$.post("/cart/remove/" + albumId, function (data) {
        $('#main').html(data); // the previous selector id ("container") was incorrect
        $('#update-message').html(albumTitle + ' has been removed from your shopping cart.');
        $cartNav.html('Cart (' + (count - 1) + ')');
    });
``` 