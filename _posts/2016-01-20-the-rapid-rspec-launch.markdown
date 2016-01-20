---
layout: post
title: The rapid RSpec launch with the custom Sublime package
description: Writing custom Sublime package to accelerate the launch of the specific spec
date: 2016-01-20 00:00:00
---
It turns out that testing is crucial. The everyday routine of good developer consists of several actions: write the failing test, implement code for this, launch the test again, refactor code and continue the process until you are satisfied.

I use Sublime as a text editor and after creating the test I always need to switch to the console, type `rspec spec/` and use autocomplete to reach the desired file. However, it is becoming irritable especially when nesting is huge. Of course, I can use `Ruby Tests` package, but it provides ability to run test only inside the text editor. I believe that there are more convenient and efficient way.

What I need is something that will generate string for the firing specific spec. For that purpose I have written a short Sublime package. In this article, I want to share how I have implemented it.

Let's start with selecting `Tools -> New Plugin` menu entry. Save the generated skeleton with name `test_runner.py` and then fill it with the following code:

```python
import sublime, sublime_plugin

class TestRunnerCommand(sublime_plugin.TextCommand):
  def run(self, edit):
    full_path = self.view.file_name() # recieve the absolute path for file
    path = full_path.split("spec/", 1)[1] # transorm it to relative path for the easy reading
    copied_string = "bundle exec rspec spec/" + path
    sublime.set_clipboard(copied_string) # save it to clipboard
```

And inside `Preferences -> Key Bindings - User` add:

```bash
[
  { "keys": ["command+shift+x"], "command": "test_runner" }
]
```

And with this, my workflow is transformed into: `cmd+shift+x` + jump to console + `cmd+v`. Thus, I got rid of the autocomplete work and free my time to make another cup of coffee. Thank you for reading!
