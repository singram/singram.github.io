---
layout: post
title: Reloading FactoryGirl definitions in a Rails 3.2 console
date: '2012-10-14 19:40:00'
tags:
- ruby
- factorygirl
- rails
---

**Problem** 

You've developed a rich set of class definitions using FactoryGirl and find them useful while developing and testing in rails console.  The problem is that when you reload! your classes the FactoryGirl definitions are not reloaded causing confusion and errors.  On top of this if, in your application, you are initializing class variables upon bootup these are lost also.  Basically adding undue weight to a simple class refresh. 

After much searching online for a solution article provided useful answers and is well worth a read. 

**Solution** 

Please note that this solution has only been tested in Rails 3.2 with FactoryGirl 4.0.0 

In `environments/development.rb`

```language-ruby
MyApplication.configure do 
  .... 
  .... 
  ActionDispatch::Reloader.to_prepare do 
    # first init will load unless  
    FactoryGirl.factories.entries.empty? 
      FactoryGirl.reload 
    end
    SomeClass.reinitialize unless SomeClass.initialized? 
  end  
end
```
[this](http://wondible.com/2011/12/30/rails-autoloading-cleaning-up-the-mess/)