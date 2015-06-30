# irbrc

My simple `~/.irbrc`

```ruby
require "irb/completion"
require "irb/ext/save-history"
IRB.conf[:SAVE_HISTORY] = 10000
require "awesome_print"
AwesomePrint.irb!
```
