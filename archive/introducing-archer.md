# Introducing Archer: Rails Console History for Heroku, Docker, and More

<p style="text-align: center;"><img src="/images/archer.png" alt="Archer" /></p>

Many companies today run infrastructure where machines or containers can be replaced at any time, so you can’t depend on them for permanent storage. One place this is especially painful is the Rails console. Console history can save a lot of typing.

This is where [Archer](https://github.com/ankane/archer) comes in. Add it your project, and it’ll begin to use the database to store history.

Archer supports multiple users so everyone on the team can have their own history. On Heroku, you can specify a user when starting the console with:

```sh
heroku run USER=andrew rails console
```

Set up an [alias](https://shapeshed.com/unix-alias/) to save some typing.

```sh
alias hc="heroku run USER=andrew rails console"
```

Add [Archer](https://github.com/ankane/archer) to your team today.
