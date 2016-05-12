# Error Reporting in R

R supports global error handling, making it easy to report all errors without individual `tryCatch` statements.

Create a file to source at the start of all your scripts.

```R
if (!interactive()) {
  options(error = function() {
    message <- geterrmessage()

    ### your error reporting goes here
    rollbar.error(message)
    ###

    write("Execution halted", stderr())
    q("no", status = 1, runLast = FALSE)
  })
}
```

Unfortunately, there’s no way to get filenames and line numbers (if you manage to do this, let me know!). Thankfully, the last line of the messages includes calls.

```txt
Error in func3(b) : unused argument (b)
Calls: func1 -> func2 -> func3
```

Happy production debugging! :dolphin:

#### If you use Rollbar...

Check out the [Rollbar](https://github.com/ankane/rollbar) package.
