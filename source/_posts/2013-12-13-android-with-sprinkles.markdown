---
layout: post
title: "Android with sprinkles"
date: 2013-12-18 00:12:27 +0100
comments: true
categories: android library tutorial
---

[Sprinkles](https://github.com/emilsjolander/sprinkles) is an android library I made because I was tired of all the boilerplate involved in dealing with SQLite on android. The goal with sprinkles is to remove as much boilerplate as possible without limiting any functionality, this is a crucial point. I find that most ORMs try to abstract away SQL which often leads to unoptimized queries and ugly hacks to get the more uncommon queries to work. Sprinkles lets you do what SQL is great for, querying data. Sprinkles will however help you with the tedious parts like inserting data and packing data out of a cursor.

Below I will give you a basic intro into using sprinkles by building a ... note taking app! Nothing new and exiting, I know! But it's a great sample app for showing the power of sprinkles. There's a lot of the api that I don't mention in this post so please head over to github and read the full documentation if you have any questions after reading this.

The app
=======
The app will be a simple note taking app where you can create multiple notes and multiple tags which you can asign one or more of to each note. Here are some screenshots of the finished app.

{% img /images/android-with-sprinkles/screenshot1.png 180 320 %}
{% img /images/android-with-sprinkles/screenshot2.png 180 320 %}
{% img /images/android-with-sprinkles/screenshot3.png 180 320 %}

I will mostly be covering the parts of the app having to do with sprinkles so I encourage you to download and play around the the full source code which you can find [here](https://github.com/emilsjolander/sprinkles/tree/master/sample).

Including the library
=====================
The easiest way to start using sprinkles is to include the library as a gradle dependecy in your build.gradle file like this.
```groovy
dependencies {
	compile 'se.emilsjolander:sprinkles:1.0.0'
}
```
The version of sprinkles used in this blog post is version `1.0.0`, remember to check the github release tab for the latest version when building your app.

If you are not using gradle (android studio defaults to using gradle) than you are probably using eclipse. If this is the case just clone the the project like this.
```bash
git clone https://github.com/emilsjolander/sprinkles.git
```
After you have cloned the library you will have to import the library folder as an android project in eclipse. Once this is done add the project as a library dependency in the project settings menu of eclipse.

Creating the models
======================
This app is going to need two main model classes, a `Note` and a `Tag`. A third model that acts as the many-to-many relational link between a `Note` and a `Tag` is also needed (sprinkles doesn't handle relationships for you. This is by design). Models must subclass the `Model` class found in the sprinkles library. Each `Model` subclass must also be annotated with a `@Table` annotation which specifies the name of the table that the model class corresponds to.

Given this information we can start creating our model classes! Lets create three empty model classes.
(I have removed the import statements to keep it short, this is something I will do in all future code samples)
```java
@Table("Notes")
public class Note extends Model {
    
}
```
```java
@Table("Tags")
public class Tag extends Model {
    
}
```
```java
@Table("NoteTagLinks")
public class NoteTagLink extends Model {
    
}
```

Lets start by adding properties to our `Note` model. A note will need an id, some content and timestamps for when it was created and last updated.
```java
@Table("Notes")
public class Note extends Model {
	
	private long id;

	private String content;
	private long createdAt;
	private long updatedAt;

}
```

Sprinkles doesn't yet know about these properties, let's fix that. To let sprinkles know that these properties correspond to fields in the database we must annotate each property with the `@Column` annotation. The `@Column` annotation takes a string parameter which is the name of the column corresponding to the annotated property. We will also add a @AutoIncrementPrimaryKey annotation to the id property. This will ensure that the models id property is a unique identifier which will automatically be assigned when a new instance is inserted into the database.
```java
@Table("Notes")
public class Note extends Model {
	
	@AutoIncrementPrimaryKey
	@Column("id") private long id;

	@Column("content") private String content;
	@Column("created_at") private long createdAt;
	@Column("updated_at") private long updatedAt;

}
```

Our `Tag` model looks very similar but with a few different properties.
```java
@Table("Tags")
public class Tag extends Model {
	
	@AutoIncrementPrimaryKey
	@Column("id") private long id;

	@Column("name") private String name;

}
```

The last model which is the `NoteTagLink` model introduces two new annotations. The @ForeignKey annotation marks the annotated column as a foreign key which references the table+column given as an argument to the annotation (See the sample code below for syntax, here note_id references the id column in the Notes table). The other new annotation is the @CascadeDelete annotation which may be used in conjunction with the @ForeignKey annotation to tell the underlying database to perform a cascade delete when the referenced foreign table row is deleted.
```java
@Table("NoteTagLinks")
public class NoteTagLink extends Model {

	@PrimaryKey
	@CascadeDelete
	@ForeignKey("Notes(id)")
	@Column("note_id") private long noteId;

	@PrimaryKey
	@CascadeDelete
	@ForeignKey("Tags(id)")
	@Column("tag_id") private long tagId;
	
}
```

We are soon finished with out models! Now we just need to add a easy way to update the `Note` models timestamps, we will do this in the `beforeCreate()` and `beforeSave()` callbacks. These callbacks are declared in the `Model` base class and may be overriden. There is also a `beforeDelete()` callback as well as a `isValid()` method that can be overriden to make sure that a model that is not valid is not saved to the database, this method is overriden by the `Tag` model in our sample app. 

Adding the above callbacks and some setters/getters to our models give us their final form.
```java
@Table("Notes")
public class Note extends Model {
	
	@AutoIncrementPrimaryKey
	@Column("id") private long id;

	@Column("content") private String content;
	@Column("created_at") private long createdAt;
	@Column("updated_at") private long updatedAt;
	
	public long getId() {
		return id;
	}
	
	public String getContent() {
		return content;
	}
	
	public void setContent(String content) {
		this.content = content;
	}
	
	public long getCreatedAt() {
		return createdAt;
	}
	
	public long getUpdatedAt() {
		return updatedAt;
	}
	
	@Override
	protected void beforeCreate() {
		super.beforeCreate();
		createdAt = System.currentTimeMillis();
	}
	
	@Override
	protected void beforeSave() {
		super.beforeSave();
		updatedAt = System.currentTimeMillis();
	}

}
```
```java
@Table("Tags")
public class Tag extends Model {
	
	@AutoIncrementPrimaryKey
	@Column("id") private long id;

	@Column("name") private String name;
	
	public long getId() {
		return id;
	}
	
	public String getName() {
		return name;
	}
	
	public void setName(String name) {
		this.name = name;
	}
	
	@Override
	public boolean isValid() {
		return name != null && !name.isEmpty();
	}
	
}
```
```java
@Table("NoteTagLinks")
public class NoteTagLink extends Model {

	@PrimaryKey
	@CascadeDelete
	@ForeignKey("Notes(id)")
	@Column("note_id") private long noteId;

	@PrimaryKey
	@CascadeDelete
	@ForeignKey("Tags(id)")
	@Column("tag_id") private long tagId;
	
	public NoteTagLink() {
		// default constructor
	}
	
	public NoteTagLink(long noteId, long tagId) {
		this();
		this.noteId = noteId;
		this.tagId = tagId;
	}
	
	public void setNoteId(long noteId) {
		this.noteId = noteId;
	}

	public void setTagId(long tagId) {
		this.tagId = tagId;
	}
	
	public long getNoteId() {
		return noteId;
	}

	public long getTagId() {
		return tagId;
	}
	
}
```

Migrating the tables
====================
Now that our models are done we need to create the underlying tables in the database, sprinkles makes this trivial! Start by creating an `Application` subclass and registering it in your `AndroidManifest.xml`. Migrating tables can be done anywhere in the app but I suggest doing it in the `onCreate()` method of an `Application` subclass. To register a custom `Application` subclass add `android:name="MY_APPLICATION_SUBCLASS"` under the application tag in your `AndroidManifest.xml`.
```java
public class MyApplication extends Application {

	@Override
	public void onCreate() {
		super.onCreate();

	}

}
```

```xml
<?xml version="1.0" encoding="utf-8"?>
<manifest xmlns:android="http://schemas.android.com/apk/res/android"
	package="se.emilsjolander.sprinkles"
	android:versionCode="1"
	android:versionName="1.0" >

	<!-- other stuff -->

	<application
		android:name=".MyApplication"
		android:allowBackup="true"
		android:icon="@drawable/ic_launcher"
		android:label="@string/app_name"
		android:theme="@style/AppTheme" >

		<!-- other stuff -->

	</application>

</manifest>
```

Time to actually migrate the tables. We will start by getting a instance of the `Sprinkles` singleton. We can then create our migration adding both out models to this migration and than adding the migration to the sprinkles object. This is the most basic of migrations but all we need for this example. Sprinkles allowes you to do all the regular migrations through easy methods and also allowes for raw sql migrations when doing more complex migrations. It is important to not change or remove a migration once an app has gone into production, otherwise sprinkles won't know which version of the database to update from/to.
```java
public class MyApplication extends Application {
	
	@Override
	public void onCreate() {
		super.onCreate();
		
		Sprinkles sprinkles = Sprinkles.getInstance(getApplicationContext());
		
		Migration initialMigration = new Migration();
		initialMigration.createTable(Note.class);
		initialMigration.createTable(Tag.class);
		initialMigration.createTable(NoteTagLink.class);
		sprinkles.addMigration(initialMigration);
	}

}
```

Listing notes
=============
[`NotesActivity`](https://github.com/emilsjolander/sprinkles/tree/master/sample/src/se/emilsjolander/sprinkles/NotesActivity.java) is responsible for listing all the notes currently in the database. It's an activity containing only a listview and an action bar item used to create a new list.

The following code executes a query returning all the notes in the database ordered by the time they were created.
```java
Query.many(Note.class, "select * from Notes order by created_at desc")
	.getAsync(getLoaderManager(), onNotesLoaded,
	NoteTagLink.class);
```
The above method calls can be divided into two parts, `Query.many()` and `getAsync()`. `Query.many()` initializes a query with a certain type of result, a `Note` in the above example. The second argument is the sql query to execute, this query could contain `?` placeholders for any vararg parameters sent as the last parameters to `Query.many()`. `getAsync()` takes three parameters in the above example. The first is the loadermanager that will manage the loader used to execute the query and the second parameter is the callback that will receive the result of the query. The last parameter (`NoteTagLink.class` is the above example) is a vararg parameter where you can pass any models that this query relies on except the queried model (`Note` in this case), we pass the `NoteTagLink` model into `getAsync()` because we want the query to be refreshed not only when a `Note` is saved/deleted but also when a `NoteTagLink` is saved/deleted. Often times you can skip this last parameter.

The callback that is called when results are loaded recieves a `CursorList` of the correct generic type. A `CursorList` is a wrapper around a cursor so you must remember to call `close()` on it at some point. Otherwise the `CursorList` is very similar to a regular `List`, you may iterate over it using javas enhanced for-loop and it has methods such as `size()` and `get(int index)`. In the callback sent to the previous query we just call `swapNotes()` on the adapter which will close the adapters previous list of results and save a reference to the new list. Notice that the callback returns a boolean, in out case `true`. This return value indicated whether we want to recieve further update in this callback if the underlying data ever changes, for listview data this is usually true.
```java
private ManyQuery.ResultHandler<Note> onNotesLoaded = 
	new ManyQuery.ResultHandler<Note>() {

		@Override
		public boolean handleResult(CursorList<Note> result) {
			mAdapter.swapNotes(result);
			return true;
		}

	};
```

Creating notes
==============
Creating notes is done in the [`CreateNoteActivity`](https://github.com/emilsjolander/sprinkles/tree/master/sample/src/se/emilsjolander/sprinkles/CreateNoteActivity.java) class. This is just a regular activity with an edittext and a button for saving the note. Below we set the `OnClickListener` of the create button. As you can see saving a note is incredibly simple, just call `save()` after setting the properties you want to set. There is also a `saveAsync()` method which saves the model in the background and a version of `saveAsync()` which takes a callback to notify you when the save has completed.
```java
findViewById(R.id.create).setOnClickListener(new OnClickListener() {
			
			@Override
			public void onClick(View v) {
				mNote.setContent(noteContent.getText().toString());
				mNote.save();
				finish();
			}
		});
```

Choosing tags
=============
I'll skip showing you how to create tags, you can check that out yourself in the [`CreateTagActivity`](https://github.com/emilsjolander/sprinkles/tree/master/sample/src/se/emilsjolander/sprinkles/CreateTagActivity.java) class, it's almost exactly like creating a note. Instead I will show you how we can set multiple tags on a note using a `NoteTagLink`, this is done in the [`ChooseTagActivity`](https://github.com/emilsjolander/sprinkles/tree/master/sample/src/se/emilsjolander/sprinkles/ChooseTagActivity.java) class. First we must query all the tags in the database. When receiving the result we swap the tags in the adapter and update what tags are checked (checked tags are attached to the note we are editing) we also return true because we want the results to update if tags are added or removed from the database.
```java
Query.many(Tag.class, "select * from Tags").getAsync(
	getLoaderManager(), onTagsLoaded);
```
```java
private ManyQuery.ResultHandler<Tag> onTagsLoaded =
	new ManyQuery.ResultHandler<Tag>() {

		@Override
		public boolean handleResult(CursorList<Tag> result) {
			mTags = result;
			mAdapter.swapTags(result);
			updateCheckedPositions();
			return true;
		}
	};
```

Next we have to query all the links between the given note and any tag, notice how we want to get updates on links when `Note` and `Tag` models are updated as well (the last arguments to `getAsync()`). When we get the results we close the old list containing links and than update the checked tags.
```java
Query.many(NoteTagLink.class,
				"select * from NoteTagLinks where note_id=?", mNoteId).getAsync(
				getLoaderManager(), onLinksLoaded, Note.class, Tag.class);
```
```java
private ManyQuery.ResultHandler<NoteTagLink> onLinksLoaded =
	new ManyQuery.ResultHandler<NoteTagLink>() {

		@Override
		public boolean handleResult(CursorList<NoteTagLink> result) {
			if (mLinks != null) {
				mLinks.close();
			}
			mLinks = result;
			updateCheckedPositions();
			return true;
		}
	};
```

Ok that wasn't too complicated! If you want to know how `updateCheckedPositions()` works just check the [`source`](https://github.com/emilsjolander/sprinkles/tree/master/sample/src/se/emilsjolander/sprinkles/ChooseTagActivity.java). There is one last thing we need to do before we are done! When a tag is clicked we need to add/remove the corrosponding `NoteTagLink` from the database. Notice how we don't update the checked status! This is because we returned true from our `ResultHandler` that queried the links, this will result in the result handler being called again when a `NoteTagLink` is saved or deleted which in turn results in the checked status of the list being updated.
```java
private OnItemClickListener onListItemClicked = 
	new OnItemClickListener() {

		@Override
		public void onItemClick(AdapterView<?> l, View v, int pos,
				long id) {
			NoteTagLink link = new NoteTagLink(mNoteId, id);
			if (mListView.isItemChecked(pos)) {
				link.saveAsync();
			} else {
				link.deleteAsync();
			}
		}
	};
```

Conclusion
==========
So that was [sprinkles](https://github.com/emilsjolander/sprinkles), I hope you like it! Please star it on github to spread the word and fork + pull request for improvements :)

The full source code for the example app can be found on [github](https://github.com/emilsjolander/sprinkles/tree/master/sample).

This was my first post ever so please comment and give me feedback on what I can improve for the next post :)
