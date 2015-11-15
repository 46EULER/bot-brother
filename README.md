# Bot-brother
Node.js library to help you easy create telegram bots. Work on top of [node-telegram-bot-api](https://github.com/yagop/node-telegram-bot-api)
Required Redis 2.8+

Main features:
  - sessions
  - middlewares
  - localization
  - templated keyboards and messages
  - navigation between commands

## Table of contents
<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->


- [Install](#install)
- [Simple usage](#simple-usage)
- [Examples of usage](#examples-of-usage)
- [Commands](#commands)
- [Middlewares](#middlewares)
  - [Predefined middlewares](#predefined-middlewares)
- [Sessions](#sessions)
- [Localization and texts](#localization-and-texts)
- [Keyboards](#keyboards)
  - [Go to command](#go-to-command)
  - [Embedded handler](#embedded-handler)
  - [isShown flag](#isshown-flag)
  - [Localization in keyboards](#localization-in-keyboards)
  - [Keyboard templates](#keyboard-templates)
  - [Keyboard answers](#keyboard-answers)
- [Api](#api)
  - [Bot](#bot)
    - [bot.api](#botapi)
    - [bot.listenUpdates](#botlistenupdates)
    - [bot.stopListenUpdates](#botstoplistenupdates)
    - [bot.command](#botcommand)
    - [bot.keyboard](#botkeyboard)
    - [bot.texts](#bottexts)
    - [Using webHook](#using-webhook)
  - [Command](#command)
  - [Context](#context)
  - [Context properties](#context-properties)
    - [context.session](#contextsession)
    - [context.data](#contextdata)
    - [context.meta](#contextmeta)
    - [context.command](#contextcommand)
    - [context.answer](#contextanswer)
    - [context.message](#contextmessage)
    - [context.bot](#contextbot)
    - [context.isRedirected](#contextisredirected)
    - [context.isSynthetic](#contextissynthetic)
  - [Context methods](#context-methods)
    - [context.keyboard(keyboardDefinition)](#contextkeyboardkeyboarddefinition)
    - [context.hideKeyboard()](#contexthidekeyboard)
    - [context.render(key)](#contextrenderkey)
    - [context.go()](#contextgo)
    - [context.goParent()](#contextgoparent)
    - [context.goBack()](#contextgoback)
    - [context.repeat()](#contextrepeat)
    - [context.end()](#contextend)
    - [context.setLocale(locale)](#contextsetlocalelocale)
    - [context.getLocale()](#contextgetlocale)
  - [context.sendMessage(text, [options])](#contextsendmessagetext-options)
    - [context.forwardMessage(fromChatId, messageId)](#contextforwardmessagefromchatid-messageid)
  - [context.sendPhoto(photo, [options])](#contextsendphotophoto-options)
  - [context.sendAudio(audio, [options])](#contextsendaudioaudio-options)
  - [context.sendDocument(A, [options])](#contextsenddocumenta-options)
  - [context.sendSticker(A, [options])](#contextsendstickera-options)
  - [context.sendVideo(A, [options])](#contextsendvideoa-options)
  - [context.sendVoice(voice, [options])](#contextsendvoicevoice-options)
  - [context.sendChatAction(action)](#contextsendchatactionaction)
  - [context.getUserProfilePhotos([offset], [limit])](#contextgetuserprofilephotosoffset-limit)
  - [context.sendLocation(latitude, longitude, [options])](#contextsendlocationlatitude-longitude-options)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->

## Install
```sh
npm install bot-brother
```

## Simple usage
```js
var bb = require('bot-brother');
var bot = bb({
  key: '<_TELEGRAM_BOT_TOKEN>',
  redis: {port: 6379, host: '127.0.0.1'}
});

// create command '/start'
bot.command('start')
.invoke(function (ctx) {
  // set data, data is used in templates
  ctx.data.user = ctx.meta.user;
  // return promise
  return ctx.sendMessage('Hello <%=user.first_name%>. How are you?');
})
.answer(function (ctx) {
  ctx.data.answer = ctx.answer;
  // return promise
  return ctx.sendMessage('OK. I understood. You are <%=answer%>');
});

// create command '/upload_photo'
bot.command('upload_photo')
.invoke(function (ctx) {
  return ctx.sendMessage('Drop me a photo, please');
})
.answer(function (ctx) {
  // ctx.message is an object that represents Message.
  // See https://core.telegram.org/bots/api#message 
  return ctx.sendPhoto(ctx.message.photo[0].file_id, {caption: 'I got your photo!'});
});

// start listen updates via polling
bot.listenUpdates();
```

## Examples of usage
We write simple notifications bot with `bot-brother`, so you can inspect code and its work here: https://github.com/SerjoPepper/delorean_bot

## Commands
Commands can set via strings and regexps.
```js
bot.command(/^page[0-9]+/).invoke(function (ctx) {
  return ctx.sendMessage('Invoked on any page')
});

bot.command('page1').invoke(function (ctx) {
  return ctx.sendMessage('Invoked only on page1');
});

bot.command('page2').invoke(function (ctx) {
  return ctx.sendMessage('Invoked only on page2');
});
```


## Middlewares
Middlewares are needed for multi stage command handling
```js
var bb = require('bot-brother');
var bot = bb({
  key: '<_TELEGRAM_BOT_TOKEN>',
  redis: {port: 6379, host: '127.0.0.1'}
})
bot.listenUpdates();

bot.use('before', function (ctx) {
  return findUserFromDbPromise(ctx.meta.user.id).then(function (user) {
    user.vehicle = user.vehicle || 'Car'
    // your can set any field name, except follow:
    // 1. fields, start with '_', like ctx._variable
    // 2. bot, session, message, isRedirected, isSynthetic, command, isEnded, meta
    ctx.user = user;
  });
});

bot.command('my_command')
.use('before', function (ctx) {
  ctx.user.age = ctx.user.age || '25';
})
.invoke(function (ctx) {
  ctx.data.user = ctx.user;
  // return promise
  return ctx.sendMessage('Your vehicle is <%=user.vehicle%>. Your age is <%=user.age%>.');
});
```
There are following stages, sorted by invoking order.

| Name         | Description                    |
| ------------ | ------------------------------ |
| before       | applied before all stages      |
| beforeInvoke | applied before invoke stage    |
| beforeAnswer | applied before answer stage    |
| invoke       | same as `command.invoke(...)`  |
| answer       | same as `command.answer(...)`  |

Lets look at following example, and try to understand how and in which order they will be invoked.
```js
bot.use('before', function (ctx) {
  return ctx.sendMessage('bot before');
})
.use('beforeInvoke', function (ctx) {
  return ctx.sendMessage('bot beforeInvoke');
})
.use('beforeAnswer', function (ctx) {
  return ctx.sendMessage('bot beforeAnswer');
});

// catch all commands
bot.command(/.*/).use('before', function (ctx) {
  return ctx.sendMessage('rgx before');
})
.use('beforeInvoke', function (ctx) {
  return ctx.sendMessage('rgx beforeInvoke');
})
.use('beforeAnswer', function (ctx) {
  return ctx.sendMessage('rgx beforeAnswer');
});

bot.command('hello')
.use('before', function (ctx) {
  return ctx.sendMessage('hello before');
})
.use('beforeInvoke', function (ctx) {
  return ctx.sendMessage('hello beforeInvoke');
})
.use('beforeAnswer', function (ctx) {
  return ctx.sendMessage('hello beforeAnswer');
})
.invoke(function (ctx) {
  return ctx.sendMessage('hello invoke');
})
.answer(function (ctx) {
  return ctx.go('world');
});

bot.command('world')
.use('before', function (ctx) {
  return ctx.sendMessage('world before');
})
.invoke(function (ctx) {
  return ctx.sendMessage('world invoke');
});
```

Bot dialog
```
me  > /hello
bot > bot before
bot > bot beforeInvoke
bot > rgx before
bot > rgx beforeInvoke
bot > hello before
bot > hello beforeInvoke
bot > hello invoke
me  > I type something
bot > bot before
bot > bot beforeAnswer
bot > rgx before
bot > rgx beforeAnswer
bot > hello beforeAnswer
bot > bot before // we jumped to "world" command with "ctx.go('world')""
bot > bot beforeInvoke
bot > rgx before
bot > rgx beforeInvoke
bot > world before
bot > world invoke 
```

### Predefined middlewares
There are following predefined middlewares
 - `botanio` - track each incoming message. See http://botan.io/
 - `typing` - show typing status before each message. See https://core.telegram.org/bots/api#sendchataction

Usage:
```js
bot.use('before', bb.middlewares.typing());
bot.use('before', bb.middlewares.botanio('<BOTANIO_API_KEY>'));
```


## Sessions
Sessions work is based on Redis 2.8+
```js
bot.command('memory')
.invoke(function (ctx) {
  return ctx.sendMessage('Type some string');
})
.answer(function (ctx) {
  ctx.session.memory = ctx.session.memory || '';
  ctx.session.memory += ctx.answer;
  ctx.data.memory = ctx.session.memory;
  return ctx.sendMessage('Memory: <%=memory%>');
})
```

following dialog demonstrates how it works:
```
me  > /memory
bot > Type some string
me  > 1
bot > 1
me  > 2
bot > 12
me  > hello
bot > 12hello
```

## Localization and texts
Localization can used in texts and keyboards.
Templates use [ejs](https://github.com/tj/ejs).
```js
// set locales
bot.texts({
  book: {
    chapter1: {
      page1: 'Hello <%=user.first_name%> :smile:'
    },
    chapter2: {
      page3: 'How old are you, <%=user.first_name%>?'
    }
  }
}, {locale: 'en'})

// set default locales (used if key in certain locale did not found)
bot.texts({
  book: {
    chapter1: {
      page2: 'How are you, <%=user.first_name%>?'
    },
    chapter2: {
      page4: 'Good bye, <%=user.first_name%>.'
    }
  }
})

bot.use('before', function (ctx) {
  ctx.data.user = ctx.meta.user // set data.user to Telegram user
  ctx.session.locale = ctx.session.locale || 'en';
  ctx.setLocale(ctx.session.locale);
});

bot.command('chapter1_page1').invoke(function (ctx) {
  ctx.sendMessage('book.chapter1.page1')
})
bot.command('chapter1_page2').invoke(function (ctx) {
  ctx.sendMessage('book.chapter1.page2')
})
bot.command('chapter2_page3').invoke(function (ctx) {
  ctx.sendMessage('book.chapter2.page3')
})
bot.command('chapter2_page4').invoke(function (ctx) {
  ctx.sendMessage('book.chapter2.page4')
})
```
When bot-brother send message, it tries interpret message as a key from your localization. If key not found, it interprets it as a template with variables and renders it via ejs.
All local variables can be set via `ctx.data`.

Texts can set for following entities:
  - bot
  - command
  - context

```js
bot.texts({
  book: {
    chapter: {
      page: 'Page 1 text'
    }
  }
});

bot.command('page1').invoke(function (ctx) {
  return ctx.sendMessage('book.chapter.page');
});

bot.command('page2').invoke(function (ctx) {
  return ctx.sendMessage('book.chapter.page');
})
.texts({
  book: {
    chapter: {
      page: 'Page 2 text'
    }
  }
});

bot.command('page3')
.use('before', function (ctx) {
  ctx.texts({
    book: {
      chapter: {
        page: 'Page 3 text'
      }
    }
  });
})
.invoke(function (ctx) {
  return ctx.sendMessage('book.chapter.page');
})
```

Bot dialog:

```
me  > /page1
bot > Page 1 text
me  > /page2
bot > Page 2 text
me  > /page3
bot > Page 3 text
```


## Keyboards
You can set keyboard for context, command or bot
```js
// this keyboard is applied for any command
// also you can use emoji in keyboard
bot.keyboard([
  [{':one: go page 1': {go: 'page1'}}],
  [{':two: go page 2': {go: 'page2'}}],
  [{':three: go page 3': {go: 'page3'}}]
])

bot.command('page1').invoke(function (ctx) {
  return ctx.sendMessage('This is page 1')
})

bot.command('page2').invoke(function (ctx) {
  return ctx.sendMessage('This is page 2')
}).keyboard([
  [{':one: go page 1': {go: 'page1'}}],
  [{':three: go page 3': {go: 'page3'}}]
])

bot.command('page3').invoke(function (ctx) {
  ctx.keyboard([
    [{':one: go page 1': {go: 'page1'}}]
    [{':two: go page 2': {go: 'page2'}}]
  ])
})
```

### Go to command
You can go to any state via keyboard. First argument for `go` method is a command name.
```
bot.keyboard([[
  {'command1': {go: 'command1'}}
]])

// or with this syntax
bot.keyboard([[
  {'command1': function (ctx) {return ctx.go('command1');}}
]])
```

### Embedded handler
You also can use embedded handler in keyboard. Important! you should not use outscope commands
```
bot.keyboard([[
  {
    'text1': {
      handler: function (ctx) {
        return ctx.sendMessage('This is my answer')
      }
    }
  }
]]);

// or with this syntax
bot.keyboard([[
  {
    'text1': function (ctx) {
      return ctx.sendMessage('Hello from there').then(function () {
        ctx.go('command1')
      })
    }
  }
]]);
```

Important! Avoid using outscope variables in embedded handler. Embedded handler functions stored in redis, so when a function will be invoked your outscope variables will not attend.
```
// This does not work! Do not use outscope variables
var outScopeText = 'some text';
bot.keyboard([[
  {
    'text1': function (ctx) {
      return ctx.sendMessage('Hello from there. ' + outScopeText).then(function () {
        ctx.go('command1')
      })
    }
  }
]]);

// This work!
bot.use('before', function (ctx) {
  ctx.data.outScopeText = outScopeText;
}).keyboard([[
  {
    'text1': function (ctx) {
      return ctx.sendMessage('Hello from there. <%=outScopeText%>').then(function () {
        ctx.go('command1');
      })
    }
  }
]]);
```

### isShown flag
`isShown` flag can be used to hide keyboard buttons in certain moment

```
bot.use('before', function (ctx) {
  ctx.isButtonShown = Math.round() > 0.5;
}).keyboard([[
  {
    'text1': {
      go: 'command1',
      isShown: function (ctx) {
        return ctx.isButtonShown;
      }
    }
  }
]]);
```

### Localization in keyboards
```js
bot.texts({
  menu: {
    item1: ':one: page 1'
    item2: ':two: page 2'
  }
}).keyboard([
  [{'menu.item1': {go: 'page1'}}]
  [{'menu.item2': {go: 'page2'}}]
])
```

### Keyboard templates
You can use keyboard templates
```js
bot.keyboard('footer', [{':arrow_backward:': {go: 'start'}}])

bot.command('start', function (ctx) {
  ctx.sendMessage('Hello there')
}).keyboard([
  [{'Page 1': {go: 'page1'}}],
  [{'Page 2': {go: 'page2'}}]
])

bot.command('page1', function () {
  ctx.sendMessage('This is page 1')
})
.keyboard([
  [{'Page 2': {go: 'page2'}}],
  'footer'
])

bot.command('page2', function () {
  ctx.sendMessage('This is page 1')
})
.keyboard([
  [{'Page 1': {go: 'page1'}}],
  'footer'
])
```

### Keyboard answers
When you want to handle text answer from your keyboard, use following code:
```js
bot.command('command1')
.invoke(function (ctx) {
  return ctx.sendMessage('Hello')
})
.keyboard([
  [{'answer1': 'answer1'}],
  [{'answer2': {value: 'answer2'}}],
  [{'answer3': 3}],
  [{'answer4': {value: 4}}]
])
.answer(function (ctx) {
  ctx.data.answer = ctx.answer;
  return ctx.sendMessage('Your answer <%=answer%>');
});
```

Sometimes you want that user manually enter the answer. Use following code to do this
```js
// Use 'compliantKeyboard' flag
bot.command('command1', {compliantKeyboard: true})
.use('before', function (ctx) {
  ctx.keyboard([
    [{'answer1': 1}],
    [{'answer2': 2}],
    [{'answer3': 3}],
    [{'answer4': 4}]
  ]);
})
.invoke(function (ctx) {
  return ctx.sendMessage('Answer me!')
})
.answer(function (ctx) {
  if (typeof ctx.answer === 'number') {
    return ctx.sendMessage('This is answer from keyboard')
  } else {
    return ctx.sendMessage('This is not answer from keyboard. Your answer: ' + ctx.answer)
  }
});
```

## Api
There are three base classes:
  - Bot
  - Command
  - Context

### Bot
Bot represents a bot.
```
var bb = require('bot-brother');
var bot = bb({
  key: '<TELEGRAM_BOT_TOKEN>',
  redis: {
    port: 6379,
    host: '127.0.0.1'
  },
  // optional
  webHook: {
    url: 'https://mybot.com/updates',
    key: '<PEM_PRIVATE_KEY>',
    cert: '<PEM_PUBLIC_KEY>',
    port: 443,
    https: true
  }
})
```

Has following methods:

#### bot.api
Api is instance of [node-telegram-bot-api](https://github.com/yagop/node-telegram-bot-api)
```js
bot.api.sendMessage(chatId, 'message');
```

#### bot.listenUpdates
Start listening updates via polling or webhook. By default `polling` is enabled.
```js
bot.listenUpdates();
```

#### bot.stopListenUpdates
Stop listen updates.
```js
bot.stopListenUpdates();
```

#### bot.command
Create command
```js
bot.command('start').invoke(function (ctx) {
  ctx.sendMessage('Hello')
});
```

#### bot.keyboard
```js
bot.keyboard([
  [{column1: 'value1'}]
  [{column2: {go: 'command1'}}]
])
```


#### bot.texts
Defined texts can use in  keyboards, messages, photo captions
```js
bot.texts({
  key1: {
    embeddedKey2: 'Hello'
  }
})

// with localization
bot.texts({
  key1: {
    embeddedKey2: 'Hello2'
  }
}, {locale: 'en'})
```


#### Using webHook
Webhook in telegram documentation: https://core.telegram.org/bots/api#setwebhook
If your node.js is running behind the proxy (nginx for example) use following code.
We omit `webHook.key` parameter and run node.js on 3000 unsecure port.
```js
var bb = require('bot-brother');
var bot = bb({
  key: '<TELEGRAM_BOT_TOKEN>',
  redis: {
    port: 6379,
    host: '127.0.0.1'
  },
  webHook: {
    url: 'https://mybot.com/updates', // your nginx from here should proxy to 127.0.0.1:3000
    cert: '<PEM_PUBLIC_KEY>',
    port: 3000,
    https: false
  }
})
```

Otherwise if your node.js server is available outside, use following code:
```js
var bb = require('bot-brother');
var bot = bb({
  key: '<TELEGRAM_BOT_TOKEN>',
  redis: {
    port: 6379,
    host: '127.0.0.1'
  },
  webHook: {
    url: 'https://mybot.com/updates',
    cert: '<PEM_PUBLIC_KEY>',
    key: '<PEM_PRIVATE_KEY>',
    port: 443
  }
})
```

### Command
```js
bot.command('command1')
.invoke(function (ctx) {})
.answer(function (ctx) {})
.keyboard([[]])
.texts([[]])
```

### Context
The context is the essence that runs through all middlewares. It is necessary to share data between middlewares. It is important to note that between middlewares passed same context, so you can record the data to be available in the next handler. Context passed as first argument in all middleware handlers.
```js
// this is handler is invoke
bot.use('before', function (ctx) {
  // ctx - it is the instance of Context
  ctx.someProperty = 'hello';
});

bot.command('mycommand').invoke(function (ctx) {
  // ctx - this is the same context!
  ctx.someProperty === 'hello'; // true
});
```

You can set any of your property to context variable. But! You must observe the following rules:
  1. Property name can not begin with an underscore. `ctx._myVar` - bad!, `ctx.myVar` - good.
  2. The names of the properties should not overlap with pre-defined properties. For example you can not reset `ctx.session` variable with something like this: `ctx.session = 'Hello'` - bad! `ctx.mySession = 'Hello'` - good.
  3. Your property must not interfere with the methods of context. See below methods of context.


### Context properties
Context has the following pre-defined properties that are available for reading. Some of them are available for editing. Let's take a look at them:
#### context.session
In this variable you can record any data, which will be available anywhere in following commands and middlewares.
Important! Currently for group chats session shares between all users in chat.

```js
bot.command('hello').invoke(function (ctx) {
  return ctx.sendMessage('Hello! What is your name?');
}).answer(function (ctx) {
  // set user answer to session.name
  ctx.session.name = ctx.answer;
  return ctx.sendMessage('OK! I remembered it.')
});

bot.command('bye').invoke(function (ctx) {
  return ctx.sendMessage('Bye ' + ctx.session.name);
});
```

The next dialog shows this:
```
me  > /hello
bot > Hello! What is your name?
me  > John
bot > OK! I remembered it.
me  > /bye
bot > Bye John
```

#### context.data
This variable works when rendering. For template rendering we use (ejs)[https://github.com/tj/ejs]. All the data you record will be available in the templates.
```
bot.texts({
  hello: {
    world: {
      friend: 'Hello world, <%=name%>!'
    }
  }
});

bot.command('hello').invoke(function (ctx) {
  ctx.data.name = 'John';
  ctx.sendMessage('hello.world.friend');
});
```

The next dialog shows this:
```
me  > /hello
bot > Hello world, John!
```

There is pre-defined method `render` in context.data. It can be used for rendering embedded keys:
```
bot.texts({
  hello: {
    world: {
      friend: 'Hello world, <%=name%>!',
      bye: 'Good bye, <%=name%>',
      message: '<%=render("hello.world.friend")%> <%=render("hello.world.bye")%>'
    }
  }
});

bot.command('hello').invoke(function (ctx) {
  ctx.data.name = 'John';
  ctx.sendMessage('hello.world.message');
});
```

Bot dialog
```
me  > /hello
bot > Hello world, John! Good bye, John
```


#### context.meta
Meta contain following fields:
  - `user` - see https://core.telegram.org/bots/api#user
  - `chat` - see https://core.telegram.org/bots/api#chat
  - `sessionId` - key name for saving session in redis, currently it is `meta.chat.id`. So for group chats your session share between all users in chat.

#### context.command
Represents currently handled command. Has following properties:
 - `name` - the name of command
 - `args` - arguments for command
 - `type` - Can be `invoke` or `answer`. If handler is invoked with `.withContext` method, type is `synthetic`

Suppose that we have the following code
```js
bot.command('hello')
.invoke(function (ctx) {
  var args = ctx.command.args.join('-');
  var type = ctx.command.type;
  var name = ctx.command.name;
  return ctx.sendMessage('Type '+type+'; Name: '+name+'; Arguments: '+args);
})
.answer(function (ctx) {
  var type = ctx.command.type;
  var name = ctx.command.name;
  var answer = ctx.answer;
  ctx.sendMessage('Type '+type+'; Name: '+name+'; Answer: ' + answer)
});
```

The result is the following dialogue:
```
me  > /hello world dear friend
bot > Type: invoke; Name: hello; Arguments: world-dear-friend
me  > bye
bot > Type: answer; Name: hello; Answer: bye
```

#### context.answer
This is answer for command. It presents only if command is a text field.

#### context.message
Presents message object. For more details look here: https://core.telegram.org/bots/api#message

#### context.bot
Shorthand for bot instance

#### context.isRedirected
Boolean. This flag means, that this command was achieved via `go` method. In other words, user did not type text `/command` in bot.
Let's look at the following example:
```js
bot.command('hello').invoke(function (ctx) {
  return ctx.sendMessage('Type something.')
})
.answer(function (ctx) {
  return ctx.go('world');
});

bot.command('world').invoke(function (ctx) {
  return ctx.sendMessage('isRedirected: ' + ctx.isRedirected);
});
```
User was typing something like this:
```
me  > /hello
bot > Type something
me  > lol
bot > isRedirected: true
```

#### context.isSynthetic
Boolean. This flag is true when we achieve the handler with `.withContext` method.
```js
bot.use('before', function (ctx) {
  return ctx.sendMessage('isSynthetic before: ' + ctx.isSynthetic);
});

bot.command('withcontext', function (ctx) {
  return ctx.sendMessage('hello').then(function () {
    return bot.withContext(ctx.meta.sessionId, function (ctx) {
      return ctx.sendMessage('isSynthetic in handler: ' + ctx.isSynthetic);
    });
  });
})
```

Dialog with bot:
```
me  > /withcontext
bot > isSynthetic before: false
bot > hello
bot > isSynthetic before: true
bot > isSynthetic in handler: true
```


### Context methods
Context has the following defined methods.

#### context.keyboard(keyboardDefinition)
Set keyboard
```js
ctx.keyboard([[{'command 1': {go: 'command1'}}]])
```

#### context.hideKeyboard()
No send keyboard
```js
ctx.hideKeyboard()
```

#### context.render(key)
Return rendered text or key
```js
ctx.texts({
  localization: {
    key: {
      name: '<%=name%>'
    }
  }
})
ctx.data.name = 'John';
var str = ctx.render('localization.key.name');
console.log(str); // outputs 'John'
```

#### context.go()
Return <code>Promise</code>
Go to some command
```js
var command1 = bot.command('command1')
var command2 = bot.command('command2').invoke(function (ctx) {
  // go to command1
  return ctx.go('command1');
})
```

#### context.goParent()
Return <code>Promise</code>
Go to parent command. The command is considered a descendant if its name begins with the parent command name, for example `setting` is parent command, `settings_locale` is a descendant command.
```js
var command1 = bot.command('command1')
var command1Child = bot.command('command1_child').invoke(function (ctx) {
  return ctx.goParent(); // go to command1
});
```

#### context.goBack()
Return <code>Promise</code>
Go to previously invoked command.
Useful in keyboard 'Back' button.
```js
bot.keyboard([[
  {'Back': function (ctx) {return ctx.goBack();}}
]])
```

#### context.repeat()
Return <code>Promise</code>
Repeat current state, useful for handling wrong answers.
```js
bot.command('command1')
.invoke(function (ctx) {
  return ctx.sendMessage('How old are you?')
})
.answer(function (ctx) {
  if (isNaN(ctx.answer)) {
    return ctx.repeat(); // send 'How old are your?', call 'invoke' handler
  }
});
```

#### context.end()
Stop middlewares chain

#### context.setLocale(locale)
Set locale for context. Use it if you use localization
```js
bot.texts({
  greeting: 'Hello <%=name%>!'
})
bot.use('before', function (ctx) {
  ctx.setLocale('en')
});
```

#### context.getLocale()
Return current locale

### context.sendMessage(text, [options])
Return <code>Promise</code>
Send text message.

**See**: https://core.telegram.org/bots/api#sendmessage

| Param | Type | Description |
| --- | --- | --- |
| text | <code>String</code> | Text  or localization key to be sent |
| [options] | <code>Object</code> | Additional Telegram query options |

#### context.forwardMessage(fromChatId, messageId)
Return <code>Promise</code>
Forward messages of any kind.

| Param | Type | Description |
| --- | --- | --- |
| fromChatId | <code>Number</code> &#124; <code>String</code> | Unique identifier for the chat where the original message was sent |
| messageId | <code>Number</code> &#124; <code>String</code> | Unique message identifier |

### context.sendPhoto(photo, [options])
Return <code>Promise</code>
Send photo

**See**: https://core.telegram.org/bots/api#sendphoto

| Param | Type | Description |
| --- | --- | --- |
| photo | <code>String</code> &#124; <code>stream.Stream</code> | A file path or a Stream. Can also be a `file_id` previously uploaded |
| [options] | <code>Object</code> | Additional Telegram query options |

### context.sendAudio(audio, [options])
Return <code>Promise</code>
Send audio

**See**: https://core.telegram.org/bots/api#sendaudio

| Param | Type | Description |
| --- | --- | --- |
| audio | <code>String</code> &#124; <code>stream.Stream</code> | A file path or a Stream. Can also be a `file_id` previously uploaded. |
| [options] | <code>Object</code> | Additional Telegram query options |

### context.sendDocument(A, [options])
Return <code>Promise</code>
Send Document

**See**: https://core.telegram.org/bots/api#sendDocument

| Param | Type | Description |
| --- | --- | --- |
| A | <code>String</code> &#124; <code>stream.Stream</code> | file path or a Stream. Can also be a `file_id` previously uploaded. |
| [options] | <code>Object</code> | Additional Telegram query options |

### context.sendSticker(A, [options])
Return <code>Promise</code>
Send .webp stickers.

**See**: https://core.telegram.org/bots/api#sendsticker

| Param | Type | Description |
| --- | --- | --- |
| A | <code>String</code> &#124; <code>stream.Stream</code> | file path or a Stream. Can also be a `file_id` previously uploaded. |
| [options] | <code>Object</code> | Additional Telegram query options |

### context.sendVideo(A, [options])
Return <code>Promise</code>
Use this method to send video files, Telegram clients support mp4 videos (other formats may be sent as Document).

**See**: https://core.telegram.org/bots/api#sendvideo

| Param | Type | Description |
| --- | --- | --- |
| A | <code>String</code> &#124; <code>stream.Stream</code> | file path or a Stream. Can also be a `file_id` previously uploaded. |
| [options] | <code>Object</code> | Additional Telegram query options |

### context.sendVoice(voice, [options])
Return <code>Promise</code>
Send voice

**Kind**: instance method of <code>[TelegramBot](#TelegramBot)</code>
**See**: https://core.telegram.org/bots/api#sendvoice

| Param | Type | Description |
| --- | --- | --- |
| voice | <code>String</code> &#124; <code>stream.Stream</code> | A file path or a Stream. Can also be a `file_id` previously uploaded. |
| [options] | <code>Object</code> | Additional Telegram query options |

### context.sendChatAction(action)
Return <code>Promise</code>
Send chat action.
`typing` for text messages,
`upload_photo` for photos, `record_video` or `upload_video` for videos,
`record_audio` or `upload_audio` for audio files, `upload_document` for general files,
`find_location` for location data.

**See**: https://core.telegram.org/bots/api#sendchataction

| Param | Type | Description |
| --- | --- | --- |
| action | <code>String</code> | Type of action to broadcast. |

### context.getUserProfilePhotos([offset], [limit])
Return <code>Promise</code>
Use this method to get a list of profile pictures for a user.
Returns a [UserProfilePhotos](https://core.telegram.org/bots/api#userprofilephotos) object.

**See**: https://core.telegram.org/bots/api#getuserprofilephotos

| Param | Type | Description |
| --- | --- | --- |
| [offset] | <code>Number</code> | Sequential number of the first photo to be returned. By default, all photos are returned. |
| [limit] | <code>Number</code> | Limits the number of photos to be retrieved. Values between 1—100 are accepted. Defaults to 100. |

### context.sendLocation(latitude, longitude, [options])
Return <code>Promise</code>
Send location.
Use this method to send point on the map.

**See**: https://core.telegram.org/bots/api#sendlocation

| Param | Type | Description |
| --- | --- | --- |
| latitude | <code>Float</code> | Latitude of location |
| longitude | <code>Float</code> | Longitude of location |
| [options] | <code>Object</code> | Additional Telegram query options |
