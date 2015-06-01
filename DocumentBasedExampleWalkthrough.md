# Creating a document-based application with Mate #

The code for this application is available at [/trunk/examples/documentbased/](http://mate-examples.googlecode.com/svn/trunk/examples/documentbased/). Both the code and this document was written by [Theo Hultberg/Iconara](http://blog.iconara.net/).

# Walk-though of the application #

## Main & ApplicationEventMap ##

The main class, defined in Main.mxml is extremely simple. It declares two things: an event map (ApplicationEventMap) and a view (MainView), that is all.

I try to do as little as possible in the main class, just the bare necessities and the things that may change if I would decide to target AIR instead of the browser, say. It should be easy to have multiple main classes targeting different platforms, but at the sama time keep code duplication to a minimum and reusability to a maximum.

Let's leave MainView to the side for the time being and focus on the ApplicationEventMap. This is the controller for the application, it wires everything together and orchestrates everything that happens. Without the application controller the code is just a bundle of unrelated classes with nothing in common.

That last sentence may sound like it was something bad. It's not, it's actually exactly what you want. The parts of your application should have as little to do with each other as possible. They should be unaware of there even being other parts. That way you can reuse them easily (in other applications, but more commonly within the application), and you can test them in isolation. The code is also easier to maintain since each part can be studied separately. The downside is that it can be harder to follow the flow within the application since communication between the parts is less direct. This is rarely a problem, but it should be said.

The ApplicationEventMap doesn't do very much by itself, but still it is the one thing that keeps the application together -- this is one of the beauties of Mate. It creates some managers, it listens for events, translates these events into actions that the managers should take, and it sets up bindings between the managers and views. It is the central hub of the application and the only thing that knows about any other parts.

By writing reusable parts that have no knowledge of the other parts of the application I can reassemble everything just by writing another event map. Instead of the monolithical style of application development promoted by, for example Cairngorm, Mate favours a style that is more akin to Lego. Just pick it apart and assemble it another way and your medieval caste is a spaceship. This is the power of loose coupling.

As I've mentioned, this example application is missing a common feature: it doesn't communicate with any backend system. If it had,  the event map would also call services and pass the results these calls to the relevant managers.

To sum up, these are the three responsibilities of an event map:

  * create managers
  * translate events into method calls on the managers, or call services and pass the results to managers
  * set up bindings between managers and views

The event map is the application controller, it is the only thing that makes the application what it is. Another application could be built by writing another event map that used the same or a subset of the parts in a different way.

## MainView & MainModel ##

The main view has a toolbar with a list of documents and a few buttons, as well as a TabNavigator. The TabNavigator is where all open documents are displayed.

If you look through the code for MainView you will see that it refers to a variable called "model" very often. This variable contains the presentation model object for the view, and is an instance of MainModel (this is my naming scheme, each view is suffixed with "View" and has a corresponding class with the suffix "Model"). The presentation model object takes care of everything that isn't strictly UI-related, it has, for example, properties that determine if the delete and close buttons, and the open document dropdown are enabled or not.

I would like to be able to say that "look how simple the view code is, that's because I use the PresentationModel pattern to keep state logic outside of the view code", but MainView isn't simple, it has a lot of code. I will get to say that a bit later when I show the DocumentView, but not right now.

The reason why MainView isn't simple is that there is some complex logic that deals with the documents that are displayed in the TabNavigator. The view has to keep track of which documents belong to which view and which of these that should be open, as well as creating new views when new documents are created. This code isn't the kind of code that you push down into the presentation model, since it's definitely UI-related (although this could definitely be debated, I have chosen to place it in the view, but placing it in the presentation model or another helper wouldn't have been wrong in any way).

One other thing of note in MainView is how it creates new views for documents. MainView has no knowledge of which kind of view to create, instead it uses a factory object for this. It gives the factory the document instance and gets a view back, to the MainView it's just a Container. This means that MainView is more or less completely decoupled from the document views, and that makes it easier to implement a feature that is not yet there, but something I would like to do: different kinds of documents.

If you have looked through the code already you may have noticed that there are two kinds of documents, plain documents and rich (text) documents. Currently these are the same, and actually only plain documents are created. The goal is to support both kinds, and this can be done without MainView having to know about it. In the future the application may support three kinds, or five, it shouldn't matter (with the exception that there must be a button or some other way for the user to say which kind he or she wants).

The toolbar in !Main view has gained complexity since I first started the application, and I will probably extract it sooner rather than later. As with any type of class I prefer to keep my views as cohesive as possible. If I can't describe what they do in a simple sentence, it's time to refactor.

## DocumentView, DocumentModel & DocumentEventMap ##

This is where things start to get interesting (and this is also where I get to say "look how simple the view code is"). DocumentView looks just like any other view, but with one difference: it declares a LocalEventMap.

LocalEventMaps are like regular EventMaps, with two differences: they only hear events from the part of the application where they are defined (i.e. the view that declared it and all its child views) and all objects they create are available only in that event map. Regular event maps hear all events in the application (all that reach the top level, for example by bubbling up), and objects they create are available to all (non-local) event maps. These things can be partly overridden, but it's true in the default case.

The reason why I have a local event map in DocumentView is that I want to handle all things having to do with a document separately from the handling of all other documents. If the user edits a document I want to keep a undo history for that document only, for example. It would require a lot of unnecessary code to make this possible using only a global application controller (i.e. event map). I would need to make an undo system that handled multiple undo histories, and all code that worked with documents would need to be aware of the possibility of there being other documents.

By doing things locally I can design a simpler undo system, because I get multiple histories automatically, each separated into it's own DocumentView-mini-application.

I have written quite extensive documentation in DocumentModel to explain how I implement the specifics of my presentation models, especially how I use bindings to achieve low coupling between the part of the code that interacts with the application (for example the setters where things are injected) and the parts that are used by the view.

When teachers of programming introduce the Model-View-Controller idea one of the main features they point to is that you can have many views showing the same data but in different ways. Usually this is illustrated by a chart and a table of numbers, both with arrows to a blob called "model". Creating a document-based application is about solving the reverse problem. You want many identical views each showing different data.

## The managers & the model ##

Managers are the model of the application. They keep track of model objects and provide a transactional interface for modifying them. By "transactional" I mean that they make sure that model objects are modified in a correct way, and perform any extra work necessary, like set a flag if an object has unsaved changes, or notifying the undo system of a new undoable action.

You may notice that my managers don't have many properties with both a getter and setter, usually only a getter. If the thing you get with the getter is settable at all, it usually has a method that can be used to update the property, but that method is more general (there is, for example a getter for open documents on ApplicationManager, but no setter, instead there is a method called `openDocument` that adds documents to the list of open documents, and so modifies the property indirectly).

There are a number of reasons for this: one is that I don't like to have public properties at all. Public properties are usually a violation of encapsulation and the information hiding principle (google ["information hiding"](http://www.google.com/search?q=information+hiding) or ["encapsulation"](http://www.google.com/search?q=encapsulation) if you're not familiar with these concepts), and as such they should be avoided.

Another reason is that it's an adaption to the way Mate works. It's easier to call methods when you want to send something to a manager, and it's necessary to have a property if you want bindings to work. Modifying your application code to suit the application framework you happen to use shouldn't be needed, but in this case I think it only enforces good behaviour. Since it doesn't make the managers dependent on Mate it's just a matter of style, and what works best.

### Mutable collections are evil ###
One thing I haven't solved very well is the fact that the ApplicationManager returns mutable collections. The `documents` getter, for example, returns a `ICollectionView` object. Even if there's no setter for this property you can modify the collection object without going through the manager. This is a serious problem that there's no good solution for in the Flex API:s. There is simply no standard collection type that is not mutable, so any time you return collections you're opening up the inner workings of an object to the caller. In other projects I try to wrap any collections I return in wrappers that throw exceptions when any destructive or modifying method is called.

You may think that that it's overkill, after all I have control over all the code and if I'm so picky about information hiding I won't abuse the fact that collections are mutable in my own code. The sad fact is that it doesn't matter if I'm careful, because some of the Flex controls modify collections you pass them, moving items, apply sorting, etc. You simply can't trust the framework with your data, so you either have to copy it before passing it on, or wrap it in an immutable wrapper.

### About commands ###
You may have learnt to encapsulate all model-modifications in commands. I think this is overdoing it. I reserve the use of commands for when I really need to encapsulate an action, for example to make it undoable, or if it is going to be executed later by some other object that doesn't need to know about the specifics of the action.

Apart from those cases I think that a simple method in a manager is good enough. The methods of the ApplicationManager class is a good example, none of them are undoable, and all are simple and direct. There is no need to stick them in commands, that would only complicate things unnecessarily.

DocumentManager, by contrast uses commands, because some of its actions are undoable (ok, so far only one actually).

### Model objects ###
It may look like there's quite a few model objects, but in reality there's only one that is important at this point: Document. There is PlainDocument and RichDocument too, but they don't do anything right now (they are part of a future expansion where the application will support multiple document types). There is also DocumentType, which just contains constants; DocumentFactory, which is the only thing that should know about how to create Document instances; DocumentData which is a value object double for Document; and finally Snapshot, which is an interface that represents a snapshot of a model object and can be safely ignored for now, it too is part of an idea that is not yet fully implemented.

### Partially immutable interfaces ###
Besides the classes and interfaces there is also a namespace called app\_internal. This namespace is part of a pattern that I haven't come up with a good name for yet (perhaps "partially immutable interface"?). The idea is to let the model objects present both a mutable and an immutable interface, one for the view tier of the application and one for the model tier.

Only managers are allowed to make changes to model objects (otherwise things like undo gets very hard to do), the view tier is only allowed to query model objects for their state. By putting all methods that modify model objects in a namespace and only use that namespace in the managers I can be fairly sure that I don't accidentally modify them anywhere else. By using the namespace a class gets access to the mutable interface, without it the objects are immutable.

### Value Objects are evil ###
You may ask yourself what the point of DocumentData is. It's a Value Object (VO) double for Document, but why have both?

To start with I'd like to say that I find the tendency of Flex developers to use VO-style objects for their model worrying. VO's are violations of encapsulation waiting to happen. The only valid use of VO's is to transfer data either from or to the backend server, or as [Parameter Objects](http://www.refactoring.com/catalog/introduceParameterObject.html) within the application. The important thing is that VO's should only be used as temporary throw-away objects, they should not be used to represent state within the model. Real objects encapsulate data, structs are what you write in C.

DocumentData is used in the application in three places: as a parameter object in the `createDocument` method of DocumentFactory, as a kind of parameter object for passing document data in DocumentEvent and for the snapshots created by the `createSnapshot` method of Document. It could be argued that the DocumentData objects created in the last example aren't temporary, since they are stored in the undo history, but at least they are treated as throw-away objects -- and besides, they are hidden behind the Snapshot interface, nothing besides Document knows that the snapshot it creates uses DocumentData.

## Some other things of note ##

### Where are my global variables? ###
If you're used to the procedural style of programming touted by frameworks like Cairngorm or PureMVC you may be wondering where the getInstance methods are. Since this is object oriented programming there aren't any. Programming with Mate works on the Hollywood Principle: "don't call us, we'll call you". Instead of objects retrieving what they need by themselves (by calling `MyFacade.getInstance()` for example) the application injects dependencies into them. This has all sorts of benefits like isolating objects from the concrete implementations of the objects they depend on. Just google for ["dependency injection"](http://www.google.com/search?q=dependency+injection) and I'm sure you will find half a million pages that will explain the benefits in great detail, or read [Miško Heverys article about how to write testable code](http://misko.hevery.com/2008/07/30/top-10-things-which-make-your-code-hard-to-test/) -- because testable code is also high quality, reusable and maintainable code.

### Undo ###
I've touched briefly on the subject on undo when discussing the UndoManager, but perhaps a more thorough explanation would be in order.

To support undo you need to be able to encapsulate each undoable action and how to reverse it in an object and save this object in an undo history. When the user want to undo you take the most recent object in the undo history and have it reverse its action. You also have to save the object in a redo history if you want to let the user redo undone actions (and you really want to do that, undo without redo makes no sense).

There are two tricky parts to implementing undo: making sure that you don't make a modification that should be undoable without puting it in the undo history, and how to make actions reversable. The way to solve the first is through good design and the other is about the Memento design pattern.

Undo is very hard to bolt on late in the development of an application. Unless you have planned for it you will have a hard time to make it work. But if you design your application so that you have strict control over where destructive modifications to the model occur implementing should be a breeze. When I say "destructive" I mean anything that changes an object -- destroying the current state, as it were.

When I discussed the managers and the model above I mentioned that I place all potentially destructive methods in their own namespace. That way I know exactly where modifications are taking place, I just have to search for that namespace. In general those places should only be in managers (and commands that those managers call). If you know where your model objects are being modified it's just a matter of encapsulating those actions in commands, and write code that reverse the actions.

In theory you could save every single modification in the undo history and reverse it by running the same action in reverse. "Add the letter 'a' at position 76 in the text" could be reversed by "remove the letter at position 76". If you try to implement this you will quickly discover that it doesn't work very well. There will be small things that you either miss, or things that your code has no control over that won't be saved, so the state will change ever so slightly between how it was when a command was executed and when it should be undone. These things add up and you will start noticing undos that return the model into an erroneous state. Either you need complete control, which isn't possible, or you need another strategy.

That strategy is the Memento pattern. You take a snapshot of an object before you modify it and save this snapshot until you are asked to undo the action. At that point you tell the object to load the snapshot, and hey presto! it will return to the previous state. No need to figure out how to do the action in reverse, you just save the previous state and reload it when needed (kind of how I play most computer games).

Now, that doesn't mean that it's easy, or that it is completely without drawbacks. The benefit of just reversing the action is that it takes no extra space, whereas taking a snapshot before each action will eat loads of memory.

When creating the snapshot you need to make sure that you create a true copy, otherwise future actions will modify your snapshot and it becomes useless. You need to make sure that the copy is a _deep copy_, that every object it references is also a copy, or a reference to an immutable object.

In this application I copy instances of the Document class by creating an instance of DocumentSnapshot, which is a private inner class of Document. It has properties for title, text and type. The first two are strings, and since strings are immutable I don't have to copy them, and the same goes for the third, which is a DocumentType object. If I had had an array or any other mutable object I would have needed to copy it and all objects that it referenced.

If you look at the DocumentSnapshot class and the Snapshot interface, don't be confused by the `xml` and `bytes` getters, they don't serve any function yet. I have an idea of extending the application so that it can save documents, and these will come into play then. For now just ignore them.

## Testing ##

I agree with the school of thought that thinks that you should be pragmatic when it comes to unit testing. I generally don't write tests before I write the actual code (although sometimes I do find it helpful) and I don't write tests for everything. The decision usually is down to whether or not the test will pay in terms of productivity, or if it will be a loss. Writing tests for getters and setters is an obvious loss, for example. It's very unlikely that you would find a bug which such a test, but you would have to pay the cost of writing it and maintaining it.

I usually don't write tests until I find the first bug that I need to use the debugger to understand and solve. At that point I start writing tests and from then on I try to maintain them. I wouldn't say it's a rule of thumb, it's more a consequence of my laziness.

In the source code you will find tests in two places: in `example.model.test` and `example.view.test`. There's no tests for the event maps, the events, the managers or the commands. I aim to write tests for the managers and the commands, but haven't seen the need yet. The event maps are incredibly hard to test, and either way they don't contain enough logic to be worth it. If you think that events should be tested you should probably stop reading right now.

The tests in `example.view.test` are probably the most interesting because of hard it generally is to test user interfaces. I've written a separate page on that topic (TestingUserInterfaces), but the short version is this: if you use the PresentationModel pattern it becomes so much easier and testing your view logic is more or less the same thing as testing any other code.

For more info check out the page about TestingUserInterfaces.