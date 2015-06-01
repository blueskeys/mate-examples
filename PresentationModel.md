# Presentation Model #

The Presentation Model pattern is an effective way of dividing the responsibilities of a view into separate parts in order to make the state logic separate from the creation of controls and the layout of the view.

The pattern was identified by Martin Fowler, and I warmly recommend reading [his description of it](http://martinfowler.com/eaaDev/PresentationModel.html). Another [very good description has been written by Paul Williams](http://weblogs.macromedia.com/paulw/archives/2007/10/presentation_pa_3.html) of Adobe Consulting.

Presentation Model is one of a number of patterns that deal with the separation of view logic and application logic, collectively called [Presentation Patterns](http://martinfowler.com/eaaDev/SeparatedPresentation.html). The idea is to "ensure that any code that manipulates presentation only manipulates presentation, pushing all domain and data source logic into clearly separated areas of the program."

There are a number of reasons to use the Presentation Model pattern: firstly the view gets simpler because most non-UI work is moved into the presentation model class. Secondly the view becomes insulated from the implementation of the application model. Thirdly the state logic is easier to test.

## Simpler view code ##

Views are most commonly written as MXML, gets simpler because much non-UI related work is moved into the presentation model class.

```
<Button label="Save" click="presentationModel.save()"/>
```

The view code can be blissfully unaware of the details of how to perform the requested action, and if the way the action needs to be performed the view doesn't need to know. The presentation model object handles that, and it's easier to change it there than in the view code.

## Testable state logic ##

Testing views is one of the hardest part of automated testing. It's very hard, sometimes almost impossible, to set up a view in isolation, or even create it in the testing environment. Instead of testing the view you can do the next best thing: test the state logic of the view.

By "state logic" I mean the code that determines if a button is enabled or not, if a checkbox is checked or not, the value of a label in a certain situation, etc.

If you let the presentation model take care of these things, and only bind to the properties of the presentation model from your view, there is not much in the view that is worth testing, and the presentation model is very simple to create in isolation.

If you are interested in this I recommend [Paul Williams articles on testing user interfaces in Flex](http://weblogs.macromedia.com/paulw/archives/2007/09/presentation_pa.html), which goes into more detail on how to test presentation model objects, as well as a number of other presentation patterns.

## Insulated views ##

Secondly the view is insulated from the application. The presentation model adapts the application model and presents it in a way that the view can use directly. Say, for example, that you have a view which displays information about a person. In the application model this information may be represented using a Person object that has a number of properties: firstName, lastName, telephoneNumber, etc. One way to display this is to give the view the Person object and let it decide:

```
<Label text="{person.firstName} {person.lastName}"/>
```

This is certainly an acceptable way of doing it but it means that the view now depends on how the application stores information about persons, if it changes the view has to be changed too -- and besides, expressions like the one above can become messy, say you want to check if there's an initial and if so add it between the first and last name with a dot after:

```
<Label text="{person.firstName} {person.initial == null ? '' : person.initial + ". "}{person.lastName}"/>
```

Another way would be to introduce a mediator (in the form of a presentation model object) between the view and the application model that did this work, and also made the view independent from the implementation of the Person class:

```
<Label text="{presentationModel.displayName}"/>
```

If the application model changed you would still have to change the code in the mediator, but that code would be more isolated and easier to change, and the view would be completely unaffected.

```
public function get displayName( ) : String {
	return person.firstName + " " + person.lastName;
}
```

Is easier to understand, and easier to change into this:

```
public function get displayName( ) : String {
	if ( person.initial == null ) {
		return person.firstName + " " + person.lastName;
	} else {
		return person.firstName + " " + person.initial + ". " + person.lastName;
	}
}
```

There is also the reverse case; when the view wants to communicate with the application. Instead calling methods or dispatching events directly from the view, the view can call methods on the presentation model object: