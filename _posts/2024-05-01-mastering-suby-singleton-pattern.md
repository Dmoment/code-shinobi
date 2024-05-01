---
layout: post
title: Mastering Singleton pattern in Ruby 
writer: Deepak Chauhan
---

### Singleton Design pattern
The singleton design pattern is a software design principle used to restrict the instantiation of a class to one single instance. This is helpful when exactly one object is needed to coordinate actions across the system. The singleton pattern ensures that a class has only one instance and provides a global point of access to it.

Here’s how it typically works:
1. ```Private Constructor```: The constructor of the class is made private to prevent other classes from creating a new instance of the class.
2. ```Private Static Instance```: The class maintains a private static instance of itself.
3. ```Public Static Method```: This method (named instance) is used by other classes to access the instance. This method checks if an instance of the class exists:
- If an instance exists, it returns this instance.
- If no instance exists, it creates one, stores it, and then returns it.

This pattern is used in various scenarios such as managing a connection to a database or the settings/preferences of an application. However, it's important to use the singleton pattern judiciously, as it can introduce global state into an application, which can complicate testing and make the code harder to understand and maintain.


In Ruby, we can achieve this by including a built-in ```Singleton``` module. Let’s understand this with an example.

```ruby
require 'singleton'

class ConfigurationManager

  include Singleton

  attr_accessor :settings

  def initialize
    @settings = laod_default_settings
  end
  def update_settings(new_settings)
    @settings.merge!(new_settings)
  end

  private
  def laod_default_settings
    {
      api_key: 'sample_key',
      end_point: 'sample_point',
      debug_mode: false
    }
  end
end

# config_manager = ConfigurationManager.new
# private method `new' called for ConfigurationManager:Class (NoMethodError)

config_manager = ConfigurationManager.instance
puts config_manager.settings

config_manager.update_settings(debug_mode: true)
puts config_manager.settings

another_config_instance = ConfigurationManager.instance
puts another_config_instance.settings

```

From the above example, you can notice:
1. Singleton Module: We include the Singleton module to ensure that only one instance of ConfigurationManager can exist.
2. Instance Variables: We store configuration settings in an instance variable that any part of the application can access through the singleton instance.
3. Dynamic Updates: We provide a method to update the settings dynamically, allowing changes at runtime that are reflected across the entire application.

In the given example, attempting to create an instance with new results in a NoMethodError because the method is private:

```ruby
config_manager = ConfigurationManager.new
# Raises NoMethodError: private method `new' called for ConfigurationManager:Class
```
To access the singleton instance of ConfigurationManager, you should use the instance method:

```ruby
config_manager = ConfigurationManager.instance
puts config_manager.settings
```

Updating this singleton instance's settings will reflect across the application since all references to ConfigurationManager point to the same instance. Even if you attempt to create what seems like a new instance:

```ruby
another_config_instance = ConfigurationManager.instance
puts another_config_instance.settings
```

You'll see that the settings have been updated globally, demonstrating the nature of the singleton pattern where all instances share the same state.

This raises further questions about how these mechanisms work under the hood, why we can't use ```new``` to instantiate the class, and why we must use ```instance``` to access the unique instance of the class. These questions can be answered by exploring the source code of the Singleton class in the official Ruby repository, available at [Ruby GitHub Repository](https://github.com/ruby/ruby/blob/master/lib/singleton.rb).

```ruby
module Singleton
  VERSION = "0.2.0"

  # Raises a TypeError to prevent cloning.
  def clone
    raise TypeError, "can't clone instance of singleton #{self.class}"
  end

  # Raises a TypeError to prevent duping.
  def dup
    raise TypeError, "can't dup instance of singleton #{self.class}"
  end

  # By default, do not retain any state when marshalling.
  def _dump(depth = -1)
    ''
  end

  module SingletonClassMethods # :nodoc:

    def clone # :nodoc:
      Singleton.__init__(super)
    end

    # By default calls instance(). Override to retain singleton state.
    def _load(str)
      instance
    end

    def instance # :nodoc:
      @singleton__instance__ || @singleton__mutex__.synchronize { @singleton__instance__ ||= new }
    end

    private

    def inherited(sub_klass)
      super
      Singleton.__init__(sub_klass)
    end
  end

  class << Singleton # :nodoc:
    def __init__(klass) # :nodoc:
      klass.instance_eval {
        @singleton__instance__ = nil
        @singleton__mutex__ = Thread::Mutex.new
      }
      klass
    end

    private

    # extending an object with Singleton is a bad idea
    undef_method :extend_object

    def append_features(mod)
      #  help out people counting on transitive mixins
      unless mod.instance_of?(Class)
        raise TypeError, "Inclusion of the OO-Singleton module in module #{mod}"
      end
      super
    end

    def included(klass)
      super
      klass.private_class_method :new, :allocate
      klass.extend SingletonClassMethods
      Singleton.__init__(klass)
    end
  end
end
```

At first glance, it may seem complex. Let's begin by understanding the order of method execution. 

So when a Singleton module is included in the class then the ```included``` method in the Singleton module will run first because the ```included``` method in the Singleton module is a callback that Ruby calls automatically whenever the module is included in another class. This is part of the Ruby module inclusion mechanism, where Ruby looks for an included method in the module being included. If it finds this method, Ruby executes it, passing the class that included the module as an argument.

We observe that the included method is defined within ```class << Singleton```. What does this unique syntax imply?

The expression ```class << Singleton``` in Ruby is used to define methods directly on the Singleton module itself, not on instances of classes that include the Singleton module. This is referred to as the "singleton class" or "metaclass" of Singleton.

In simpler terms, when methods are defined within the ```class << MyModule``` block, they become class-level methods exclusively for ```MyModule```, not for classes that include MyModule. These methods are accessible on MyModule similarly to static methods in other programming languages. If you include MyModule in another class, the methods from the ```class << MyModule``` block do not become available as class methods in the including class. Instead, only the instance methods defined outside this block in MyModule are mixed into the including class.

Let’s understand this with an example - 

```ruby
module MyModule
  def instance_method
    'instance method'
  end
end

class << MyModule
  def class_method
    'class method'
  end
end

class MyClass
  include MyModule
end

puts MyModule.class_method          # Outputs: class method
puts MyClass.new.instance_method    # Outputs: instance method
# MyClass.class_method             # This would raise a NoMethodError because class_method is not available on MyClass
```

In this example, MyModule.class_method is accessible directly through MyModule, but it is not available as a method on MyClass or any instances of MyClass. Only instance_method becomes part of MyClass through the include.

Now, getting back to our included method.

```ruby
def included(klass)
  super
  klass.private_class_method :new, :allocate
  klass.extend SingletonClassMethods
  Singleton.__init__(klass)
end
```

In the context of the Singleton module, the included method is used to set up the singleton pattern for the class that includes it. This involves:
1. Making ```new``` and ```allocate``` private in the including class to prevent external instantiation.
2. Extending the including class with additional class methods (from ```SingletonClassMethods```).
3. Initializing singleton-related instance variables and methods in the including class.


The ```Singleton.__init__(klass)``` method is called to initialize the class that includes the Singleton module. Here's what it does:
Initialize Singleton Instance Variable: It sets ```@singleton__instance__``` to nil. This variable is used to hold the singleton instance once it's created, ensuring that only one instance exists.
Initialize Mutex: It also sets up ```@singleton__mutex__``` using Thread::Mutex.new. This mutex is used to synchronize the creation of the singleton instance to ensure thread safety, meaning that the singleton instance is created only once even in a multi-threaded environment.

```ruby
def __init__(klass) # :nodoc:
  klass.instance_eval {
    @singleton__instance__ = nil
    @singleton__mutex__ = Thread::Mutex.new
  }
  klass
end
```

By using ```instance_eval```, these variables are set directly on the class itself, not on instances of the class. This setup is part of ensuring that the class adheres to the singleton pattern, managing its own single instance and providing thread-safe access to it.

We extended ```klass.extend SingletonClassMethods```. This action makes all the methods in the SingletonClassMethods module become class methods for the class that includes this module. For example, in our previous scenario where we accessed the sole instance of the class using ```ConfigurationManager.instance```, this method call was possible because of these extensions.

It actually calls the instance method - 
```ruby
def instance # :nodoc:
  @singleton__instance__ || @singleton__mutex__.synchronize { @singleton__instance__ ||= new }
end
```

Here, it simply checks if a singleton_instance already exists. If it does, it returns that instance; otherwise, it creates a new one and returns it. This ensures that only one instance is available at all times.

Also, ```clone and dup```: Both methods are overridden to raise TypeError when trying to clone or duplicate the singleton instance, ensuring that the singleton's uniqueness is maintained.