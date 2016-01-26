Bundler
============

[![Build Status](https://travis-ci.org/workarounds/bundler.svg?branch=master)](https://travis-ci.org/workarounds/bundler)

Generates broilerplate code for intent and bundle builders and parsers. Autogeneration of this code at compile time ensures type-safety.
Here's an example of this in Action.

```java
@RequireBundler
class BookDetailActivity extends Activity {
  @Arg @State
  int id;
  @Arg @State
  String name;
  @Arg @Required(false) @State
  String author;

  @Override 
  public void onCreate(Bundle savedInstanceState) {
    super.onCreate(savedInstanceState);
    setContentView(R.layout.activity_book_detail);
    Bundler.inject(this);
    // TODO Use fields...
  }
  
  @Override
  protected void onRestoreInstanceState(Bundle savedInstanceState) {
    super.onRestoreInstanceState(savedInstanceState);
    Bundler.restoreState(this, savedInstanceState);
  }

  @Override
  protected void onSaveInstanceState(Bundle outState) {
    super.onSaveInstanceState(outState);
    Bundler.saveState(this, outState);
  }
}
```

After defining the annotating the activity methods are added to the 'Bundler' class which help in building and parsing the intent for the activity. The above activity can be started as follows:

```java
  Bundler.bookDetailActivity(1, "Hitchhiker's guide to galaxy")
    .author("Douglas Adams")
    .start();
```

Two classes are generated by the annotation processor. One `Bundler` class is generated which has the `inject`, `restoreState` and `saveState` methods of all annotated classes. And one `BookDetailActivityBundler` class is generated for `BookDetailActivity`. Here are the generated classes: [BookDetailActivityBundler](https://github.com/workarounds/bundler/wiki/BookDetailActivityBundler), [Bundler](https://github.com/workarounds/bundler/wiki/Generated-Bundler-class) 

As you can see defining intent keys and parsing intents is not needed anymore. See [Why Bundler?](https://github.com/workarounds/bundler/wiki/Why-Bundler%3F) for a detailed explanation. State is also saved to bundle and retrieved backed.

If in future if the field `id` in `BookDetailActivity` for some reason has to be changed to type `String` then the class `Bundler` is regenerated and all the places where an `int` is being passed to the `BookDetailActivity` will throw a compile time error compared to the run time error it would have lead to in the normal scenario.
The process for annotating Fragments and service is similar, but instead of `.start()` method fragment's builder will have `.create()` method.
Here's an example for a fragment

```java
@RequiresBundler
class BookDetailsFragment extends Fragment {
  @Arg 
  int id;
  @Arg 
  String book;
  @Arg @Required(false)
  String author;
  
  @Override
  public View onCreateView(LayoutInflater inflater, ViewGroup container, Bundle savedInstanceState) {
    Bundler.inject(this);
    // TODO inflate and return the view and use the fields
  }
}
```

The above fragment can be created as follows:

```java
BookDetailsFragment fragment = Bundler.bookDetailsFragment(1, "Harry Potter")
                                  .author("J. K. Rowling")
                                  .create();
```
This would create a `BookDetailsFragment` that have arguments set to the above values.

The process of documenting and writing tests is on going. The library is currently being used in our app [Define](https://play.google.com/store/apps/details?id=in.workarounds.define) and
another library [Portal](https://github.com/workarounds/portal). Any PRs, suggestions are more than welcome. 


Download
--------
Gradle:
```groovy
buildscript {
  dependencies {
    classpath 'com.neenbedankt.gradle.plugins:android-apt:1.8'
  }
}

apply plugin: 'com.neenbedankt.android-apt'

dependencies {
  compile 'in.workarounds.bundler:bundler-annotations:0.0.3'
  apt 'in.workarounds.bundler:bundler-compiler:0.0.6'
}
```

What can be an `@Arg` or `@State`?
----------------------------------
The short answer is anything that can be put into a `Bundle`. All primitives, `String`, any object that implements `Parcelable`, `Serializable`, `ArrayList<Parcelable>`, `ArrayList<Object>` (as `ArrayList` implements `Serializable`) can all be annotated with `@Arg` or `@State`. Currently `List` cannot be annotated, maybe in a future release we'll internally cast it to an `ArrayList` if it's implemented that way and support it. See [#2](https://github.com/workarounds/bundler/issues/2)

Additional options to `@RequireBundler`
--------------------------------------
The `@RequireBundler` annotation accepts four arguments:
* **requireAll (boolean defaults to true)** when set to true all fields are assumed to be as required unless specified using `@Required(false)`, similarly when set to false all fields are assumed optional unless specified using `@Required(true)`. This is there just for convenience if you have more fields that are required set this to `true` and if you have more optional fields set this to `false`

* **bundlerMethod (String defaults to "")** this is to specify the name of the method that corresponds to this class in the generated `Bundler` class. The above `BookDetailActivity` by default generates a method `Bundler.bookDetailActivity(...)`, but you can specify a different name by annotation it with `@RequireBundler(bundlerMethod="detailActivity")` this generates the method `Bundler.detailActivity(...)` which can be used to start the activity. This is also useful when there are two activities with the same name. If there are two `BookDetailActivity`s in the project you have to specify a different name for atleast one of them.

* **inheritArgs (boolean defaults to true)** If the super class of the current class is also annotated with `@RequireBundler` and the super class also contains fields annotated with `@Arg` then those fields will also be considered as arguments of current class. Discussed further below.

* **inheritState (boolean defaults to true)** Similar to `inheritArgs`. Fields in the super class annotated with `@State` are also saved and restored in the subclass. 

Inheritence
-----------
For example, consider the following classes `BaseActivity` and `ChildActivity`

```java
@RequireBundler(requireAll=false)
public class BaseActivity extends Activity {
  @Arg
  int first;
  @Arg @Required(true)
  int second;
  @Arg @Required(false)
  int third;
  
  // rest of the class
}

@RequireBundler(inheritArgs=true)
public class ChildActivity extends BaseActivity {
  @Arg
  String fourth;
  
  // rest of the class
}
```

The above code leads to two generated methods in `Bundler` that can be used as below:

```java
// passing first and third values is optional
Bundler.baseActivity(2).first(1).third(3).start(ctx);
// only the value of field second needs to be passed to start the activity
Bundler.baseActivity(2).start(ctx);

// passing third is optional
Bundler.childActivity(1, 2, "4").third(3).start(ctx);
// where as first, second and fourth are required to start the activity
Bundler.childActivity(1, 2, "4").start(ctx);
```
In the above code the activities are started by passing the values `first = 1`, `second = 2`, `third = 3`, `fourth = "4"` and `ctx` is `Context` object.

Few things to note here are:
* the order of fields in the generated method is parent fields followed by child fields in the order they are defined.
* the global `requireAll` behavior is not inherited, where as the field-wise `@Required` behavior is inherited. So if it's required that `ChildActivity` does not require the field `second` to be necessarily passed in the intent then simply redefine that field in `ChildActivity` as follows:

```java
// inheriArgs is true by default
@RequireBundler
public class ChildActivity extends BaseActivity {
  @Arg @Required(false)
  int second;
  @Arg 
  String fourth;
  
  // rest of the class
}
```

`ChildActivity` can now be started as
```java
// passing second and third is optional
Bundler.childActivity(1, "4").second(2).third(3).start(ctx);
// only first and fourth are required fields
Bundler.childActivity(1, "4").start(ctx);
```

License
-------

    Copyright 2015 Workarounds

    Licensed under the Apache License, Version 2.0 (the "License");
    you may not use this file except in compliance with the License.
    You may obtain a copy of the License at

       http://www.apache.org/licenses/LICENSE-2.0

    Unless required by applicable law or agreed to in writing, software
    distributed under the License is distributed on an "AS IS" BASIS,
    WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
    See the License for the specific language governing permissions and
    limitations under the License.


