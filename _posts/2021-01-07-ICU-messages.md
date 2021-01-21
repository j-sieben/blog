# ICU (International Components for Unicode) support for messages added to `PIT`

It was a feature that was on my todo list for quite a long time. I didn't implement it yet because a Java library would be necessary to implement this functionality. I may be done in pure PL/SQL, but this would have been a project much to big for a single developer. Java, on the other hand, wasn't available in all databases of Oracle, so I felt it was no real option to use Java as this would exclude some users. With the advent of the Oracle XE version 12 the last database without a JVM is gone, so this argument is gone.

## What is ICU and how does it relate to `PIT`?

### What is ICU?

ICU is a large standard around internationalization and Unicode, most of which is of no real relevance to `PIT`. But there is a section within ICU that deals with formatting messages, especially with the requirement of internationalization in mind. The problems ICU addresses in this regard are known to any developer. Let's start with a simple example:
```
#0# file(s) loaded succesfully.
```
This type of message is so common for developers and users of software alike that it's even hard to point out the problem here. Obviously, it would be nicer to have a message along the lines of:
```
One file loaded succesfully.
Two files loaded successfully.
3 files loaded successfully.
```
But this would mean that you either have to provide many messages and choose one of them depending on the amount of files loaded, or to glue your message text together yourself. The latter version is even worse because this solution is almost impossible to translate to other languages. There is a subtle additional problem to this message: Other countries may have different rules on how to express that no, one, two, three or more files were loaded. You may even don't know about those subtle differences, making it impossible for you to cater for these changes.

To solve these issues, ICU messages allow for conditionally enriched messages. Instead of implementing the respective code within the application, the message itself takes care of selecting the correct form. This then enables a translator to adopt the message text to the traditions and grammar of the respective language, taking away that burdon from the developer.

To achieve this, a message needs to have a contract that describes which information is required to generate correct messages in different languages. It may be that in one scenario the gender of the acting person is required, more often it may depend on a number of events that occurred. If we continue with our example, a solution to the problem could be to create a message like this:
```
{num_of_files, plural, offset:1
  =0 {No file loaded.}
  =1 {One file loaded successfully.}
  =2 {Two files loaded successfully.}
  other {# files loaded succesfully.}}
```

If you pass this message along with the amount of files loaded to ICU, it will choose the correct message. The beauty of this approach is: Should a translator feel that there has to be a different message for up to three messages and only then the other tree, she can do so by simply adding another branch to the select tree.

For `PIT`, this comes in handy. When I examined the requirements of ICU and the environment of `PIT`, there are two challenges to solve:
- How does `PIT` tell message in ICU format from normal `PIT` messages?
- How can we pass named anchors to `PIT`

### How to tell ICU messages from `PIT` messages

I didn't want to add a parallel API just to support ICU messages, therefore I needed to find a way to pass this information to `PIT` within the boundaries of the existing API. I decided to separate the messages by an easy to remember convention: If you want `PIT` to treat a message as an ICU message, you must pass in the string `FORMAT_ICU` (or the constant `pit.FORMAT_ICU`) as the first parameter for `P_MSG_ARGS`. The second parameter contains all named anchors and their values for that message:

```
pit.print(msg.MY_ICU_MESSAGE, msg_args(pit.FORMAT_ICU, '<ICU parameters>'));
```

### How to pass named anchors to a message

`PIT` handles the message anchors internally by their index. I felt that this wouldn't be a solution for ICU messages because decision logic is based on these anchors. Having a message that says `{0, select, =Y{some message} =N{some other message}}` is useless for a translator, especially, if it gets more complicated.

A natural choice for me was to pass those named anchors and values as a JSON instance, especially because ICU expects number values not to be passed as string values. This way, I could stick to the existing API and reserve the second parameter of `P_MSG_ARGS` for the JSON string with the anchor names and values. Keep in mind though that `MSG_ARGS` is a table of CLOB. You can't pass in a JSON object but only its string representation.

## What's the benefit for `PIT`?

It is now even easier to write code that meets the requirements of internationalisation. The ability to use ICU messages increases the overall quality of the code. As these messages are seamlessly integrated into the existing API, you can decide for each individual message whether you need the additional functionality of ICU messages or whether the easier-to-use normal `PIT` message is sufficient.

Read more about this concept and how to include it into `PIT` [here](https://github.com/j-sieben/PIT/blob/master/Doc/icu_messages.md).

## How the ICU extension for `PIT` works

It turned out that the required changes to `PIT` weren't as massive as feared. After importing the required ICU libraries (see the next section) and the small Java wrapper to call it from PL/SQL, it turned out that adding this wrapper method as a static member function to the `MESSAGE_TYPE` was the only change. In the constructor method of this type, it is analyzed whether the first parameter is `pit.FORMAT_ICU` and if it is, the static wrapper method is called to format the message. This was a nice proof of my concept to provide an »intelligent« message type rather than a simple string. Adding ICU was a snap basically and it leaves room for similar extension in the future.

As ICU is not in integral part of the Oracle database, it is necessary to pass the actually set locale from the session context as an explicit parameter. Adding this to the Java library would have been possible but it would incur additional complexity by setting up an internal database connection from Java. Passing this information as a parameter was considered the simplest possible implementation.

## The ICU class

Luckily, there is an officially supported implementation of ICU for Java available at the [ICU project site](http://site.icu-project.org/download). I selected the actual version (68.2 at the time of this writing) and loaded it into the database. Additionally, I required support for JSON to be able to parse the named parameter provided in this format. To cater for this, I added `org.json` to the game. It can be downloaded [here](https://jar-download.com/artifacts/org.json).

What is required then is a trivial Java project (well, at least for a seasoned Java developer, that is ... Unfortunately, I do not count myself among this group of people, so sorting out the Java related issues took a little time for me). Nevetheless, the simplest solution I could come across with is this method:

```
package icu;

import java.util.Locale;
import java.util.Map;
import com.ibm.icu.text.MessageFormat;

public class ICU {

  public static final int SUCCESS = 0;
  public static final int FAILURE = 1;

  public static String format(String message, String JSONParams, String ActiveLocale, int[] StatusCode, String[] ErrorMessage) {
      String formattedMessage = message;
      Locale locale = new Locale(ActiveLocale);
      MessageFormat formatter = new MessageFormat(message);
      formatter.setLocale(locale);
      try {
          Map <String, Object> attributes = JsonUtils.JSONStringtoMap(JSONParams);
          try {
              formattedMessage = formatter.format(attributes);
          }
          catch(IllegalArgumentException e){
              StatusCode[0] = FAILURE;
              ErrorMessage[0] = e.getMessage();
          }
      }
      catch(JSONException e){
           StatusCode[0] = FAILURE;
           ErrorMessage[0] = JSONParams + ": " + e.getMessage();
      }
      return formattedMessage;
  }

```

In regard to the JSON parameters, it was necessary to convert the JSON structure to the object map ICU expects as a parameter format. I've done this using a little helper class. Java stored procedures within the Oracle database are a bit thorny when it comes to exception handling, therefore this somewhat unusual approach of passing a status and an error message as out parameters was chosen. Should anybody know a more elegant way of dealing with exceptions, please let me know.
