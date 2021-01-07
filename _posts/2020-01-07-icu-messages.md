# ICU (International Components for Unicode) support for messages added to `PIT`

It was a feature that was on my todo list for quite a long time. I didn't implement it yet as I felt that a Java library would be necessary to implement this functionality. I may be done in pure PL/SQL, but this would have been a project much to big for a single developer. Java, on the other hand, wasn't available in all databases of Oracel, so I felt it was no real option to use Java as this would exclude some users. With the advent of the Oracle XE version 12 the last database without a JVM is gone, so the arguments agains using this functionality is gone.

## What is ICU and how does it relate to `PIT`?

### What is ICU?

ICU is a large standard around internationalization and Unicode, most of which is of no real relevance to `PIT`. But there is a section within ICU that deals with formatting messages, especially with the requirement of internationalization in mind. The problems ICU addresses are known to any developer. Let's start with a simple example:
```
#0# file(s) loaded succesfully.
```
This type of message is so common for developers and users of software alike that it's even hard to point the problem here. Obviously, it would be nicer to have a message that says
```
One file loaded succesfully.
Two files loaded successfully.
3 files loaded successfully.
```
But this would mean that you either have to provide many messages and choose one of them depending on the amount of files loaded, or to glue your message text together yourself. The latter version is even worse than the first because this solution is almost impossible to translate to other languages. There is a subtle additional problem to this message: Other countries may have different rules on how to express that no, one, two, three or more files were loaded. You may even don't know about those subtle differences, making it impossible for you to cater for these changes.

To solve thos issues, ICU messages allow for conditionally enriched messages. Instead of implementing the respective code within the application, the message itself takes care of selecting the correct form. This then enables a translator to adopt the message text to the traditions and grammar of the respective language, taking away that burdon from the developer.

To achieve this, a message needs to have a contract that describes, which information is required to generate correct messages in different languages. It may be, that in one scenario the gender of the acting person is required, more often it may depend on a number of events that occurred. If we continue with our example, a solution to the problem could be to create a message like this:
```
{num_of_files, plural, offset:1
  =0 {No file loaded.}
  =1 {One file loaded successfully.}
  =2 {Two files loaded successfully.}
  other {# files loaded succesfully.}}
```

If you pass this message along with the amount of files loaded to ICU, it will generate the correct message. The beauty of this approach is: Should a translator feel that there has to be a different message for up to three messages and then the other tree, she can by simply adding a third branch to the select tree.

For `PIT`, this comes in handy. When I examined the requirements of ICU and the environment of `PIT`, there are two challenges to solve:
- How does `PIT` tell message in ICU format from normal `PIT` messages?
- How can we pass named anchors to `PIT`

### How to tell ICU messages from `PIT` messages

I didn't want to add a parallel API just to support ICU messages, therefore I needed to find a way to pass this information to `PIT` within the boundaries of the existing API. I decided to separate the messages by an easy to remember convention: If you want `PIT` to treat a message as an ICU message, you must pass in the string `FORMAT_ICU` (or the constant `pit.FORMAT_ICU`) as the first parameter for `P_MSG_ARGS`. The second parameter contains all named anchors and their values for that message.

### How to pass named anchors to a message

`PIT` handles the message anchors internally by their index. I felt that this wouldn't be a solution for ICU messages because decision logic is based on these anchors. Having a message that says `{0, select, =Y{} =N{}}` is useless for a translator, especially, if it gets more complicated.

A natural choice for me was JSON to pass those anchor names and values across, especially because ICU expects us to separate number, date and string values. This way, I could stick to the existing API and reserve the second parameter of `P_MSG_ARGS` for the JSON string with the anchor names and values.

## What's the benefit for `PIT`?

It is now even easier to write code that meets the requirements of internationalisation. The ability to use ICU messages increases the overall quality of the code. As these messages are seamlessly integrated into the existing API, you can decide for each individual message whether you need the additional functionality of ICU messages or whether the easier-to-use normal `PIT` message is sufficient.


