---
layout: post
title: 「中文翻译」 如何使 App 在离线状态下保持工作
categories: 个人译文
tags: Translations
date: 2017-07-12 22:23:24 +8000
---

> 原文来源：[How to Make Mobile Apps Work Offline](https://dzone.com/articles/how-to-make-mobile-app-work-offline)
作者: [Yura Bondarenko](https://dzone.com/users/2993304/ybondarenko.html)

前言：你应该让你的移动应用尽可能地容易上手，那就让我们来看看在针对离线功能的应用开发中出现的挑战和一系列的解决方案。<!-- more -->

在理解如何构建能灵活地处理不同网络场景下离线模式的应用之前，知道最主要能帮你编写出支持离线模式应用的核心技术是很有必要的。要让一款应用提供离线支持，通常有两个比较核心的能力做基础：

* 本地持久化存储；
* 数据的同步性。

让我们仔细看看在开发过程中有可能会出现的各种问题，以及深入了解以上两个能帮助我们处理离线应用的功能点。

## 离线数据的存储和使用

为了保证应用在离线模式下的正常工作，你应当时刻需要将数据保存到客户端设备上。这样做的话，即便在丢失网络连接时，你的应用都可以非常有效率地工作。这里有几种离线数据的存储方式能使一个应用正常运行于离线环境中。当然，不同平台（iOS、Android、Windows phone 等）有着不同的处理方法，我们在接下来会一一分析它们。

### 本地缓存和 Cookies

当你使用 web 技术去搭建一个移动应用时，很可能就接触到浏览器的缓存或 cookies。通常来说，当你访问一个具体的 URL 时，你的浏览器会给服务器发送一个请求，然后会返回一个合适的页面，但是在脱机工作时，浏览器就无法展示之前请求的页面了。在这种情况下，你可能就会需要一个缓存清单文件，也许只是一个简单的必需的文件列表。将设置完毕的缓存清单告知浏览器，让浏览器知道它接下去该如何处理那些已经下载完毕的页面，而不是在无网络连接时马上去显示一个错误。

**缓存清单文件：**
```
style.css
jquery-3.2.1.min.js
index.html
```

应当将这份缓存清单文件的 MIME 设置成 *text/cache-manifest* 类型，并在 HTML 文件中引入它：
```html
<html manifest="offline.manifest">
```

当浏览器加载文件时，它会询问你是否允许将数据缓存至你的设备中。通过这样的方法，即使失去了网络连接之后，还能允许基于网络的移动应用继续工作。比如说，即使没有连接到无线热点，同事们还是可以阅读新闻和邮件信息。

另一种比较常见的方法是，使用本地保留的应用数据，也就是浏览器 cookies。这是一种最基础也比较受限制的方法，因为 cookies 大约只能保存 4KB 大小的数据。这种方法的缺点之一是 cookies 会重新发送每一个 HTTP 请求到服务器，这样的话，即使你不需要的离线数据，它也会去重新请求，这种结果会导致大量的带宽被浪费。你需要记住的一点就是，cookies 是很容易被篡改的，即使用户只是单纯地想清除 cookies，所以这种方法只会在被要求存储简单的数据的时候才会被用到。

### 共享偏好设置

要为 Android 和 iOS 开发支持离线功能的应用，你需要一种必然的机制，这种机制，能允许用户设置自己的存储方式，并使用它能配置自己的应用。通常地，移动应用会暴露出一些能够让用户进行自定义界面和行为的配置。

#### Android 平台

Android 提供了一种名为 [SharedPreferences](https://developer.android.com/reference/android/content/SharedPreferences.html) 的持久化方式，它所提供的 API 能够以键值对的形式来存储一些相对而言比较简单的原始数据。简而言之，一个 *SharedPreferences* 对象，是一个包含了一个或多个键值对文件的引用，且对象提供了简单的读写操作接口。对于这种对象，你只能采用字符类型作为它的键，并且为这个键提供合适的值。再说说对应的值，它可以接受一下几种类型的一种：*boolean*、*float*、*int*、*long* 和 *string*。每一个 *SharedPreferences* 文件都会在 Android 系统内部进行管理，本质上来说，它就是存储在内部目录中的一个 .xml 文件。一个应用可以有多个这样的文件，在理想情况下，它们被用作存储应用的一些设置。之前提到的 API 还提供了两个用于获取 *SharedPreferences* 对象的上下文方法。在应用中只有单个这样的对象文件时，这两个中的第一个方法就会被用到；而第二个方法则会在从多个对象文件中取出一个时用到，或者你已经指定了一个自定义的文件名，用于获取这个特定的对象文件时。下方的示例就是 *SharedPreferences* 在 Android 中的基本用法：

```java
SharedPreferences sharedPreferences = getPreferences(MODE_PRIVATE);
SharedPreferences sharedPreferences = getSharedPreferences(fileNameString, MODE_PRIVATE);
```

如果要添加或者修改一个值，可以使用 *Editor* 属性的 *putXXX()* 方法，「XXX」指代上述值类型的一种具体的名称。你也可以通过调取 *remove()* 方法去移除一对键值。最后，调用 *Editor* 的 *commit()* 方法，以确保改动生效。以下是对 *SharedPreferences* 的一些管理操作：

```java
SharedPreferences.Editor editor = sharedPreferences.edit();
editor.putString(keyString, valueString);
editor.commit();
```

#### iOS 平台

同样的，为一个 iOS 应用开发离线模式，你可以使用 *NSUserDefaults* 类来保存和更新用户数据。这个类提供了一些能够允许应用去更改用户持久数据的程序接口。举例来说，你可以给用户提供一个离线保存个人信息头像的方式，或者提供一种自动保存文档的特性。这些保存在应用中的数据在 iOS 平台中被称为[「Defaults System」](https://developer.apple.com/library/content/documentation/Cocoa/Conceptual/UserDefaults/AboutPreferenceDomains/AboutPreferenceDomains.html)（注：链接参阅「About the User Defaults System」）。使用这种方法将信息缓存了之后，就避免了用户每次保存一个值时都会去调用一次系统数据库这种行为。而且随着iOS的 「Defaults System」在应用的所有代码中都能够使用，保存到「Defaults System」的任何内容都将通过应用程序会话存留。这对于用户来说，无论他是关闭了应用还是重启了他的手机，他依然可以在下一次打开应用后保存他想保存的数据。

#### 本地（包括内部和外部）的存储

在很多情况下，上述提到的例如 *SharedPreferences* 所提供的方法限制较多，可能无法满足你的所有需求，比如存储图片、序列化对象、存储 .xml，JSON 或者其他文件。那么你就需要内存缓存或者外置的存储机制来帮助你。当你需要将数据存储在设备的文件系统中时，它可以满足上述特定的存储需求，并且并不依赖于关系型数据库所提供的功能。除此之外，使用存储器可以非常快速地去操作数据，并且使用方式十分简单。另外，将数据通过内存方法来存储于设备中，这些数据在应用中是完全保密、非常安全的，当应用被卸载后，那些数据也会在设备上被抹掉。

#### 使用 SQLite 数据库

最后，在大多数移动平台上，类似于 Android，iOS 和 Windows Phone，尽管不同的平台有着不同的数据库管理方法，不过都为构建其上的应用提供了 *SQLite* 的数据库作为持久化工具。这就是为什么这个数据库我们需要放在最后分析。SQLite 是一个开源的数据库系统，它为应用提供了快速的、强大的并且功能完整的关系型数据库的功能。为了使得管理起来相对简单，SQLite 使用单个文件来管理和存储所有数据。当然，它不会在同步和冲突解决方面处理太多事务，但是它在队列和缓存数方面提供了多种简单易用的方式。所以如果你需要查询存储的数据，SQLite 的持久化方案是一个值得考虑的备选项。以下是一些有关如何在 Android 中创建一个数据库表的代码样例：

```java
public class SampleSQLiteDBHelper extends SQLiteOpenHelper {
    private static final int DATABASE_VERSION = 2;
    public static final String DATABASE_NAME = "sample_database";
    public static final String PERSON_TABLE_NAME = "person";
    public static final String PERSON_COLUMN_ID = "_id";
    public static final String PERSON_COLUMN_NAME = "name";
    public static final String PERSON_COLUMN_AGE = "age";
    public static final String PERSON_COLUMN_GENDER = "gender";
    public SampleSQLiteDBHelper(Context context) {
        super(context, DATABASE_NAME, null, DATABASE_VERSION);
    }
    @Override
    public void onCreate(SQLiteDatabase sqLiteDatabase) {
        sqLiteDatabase.execSQL("CREATE TABLE " + PERSON_TABLE_NAME + " (" +
                PERSON_COLUMN_ID + " INTEGER PRIMARY KEY AUTOINCREMENT, " +
                PERSON_COLUMN_NAME + " TEXT, " +
                PERSON_COLUMN_AGE + " INT UNSIGNED, " +
                PERSON_COLUMN_GENDER + " TEXT" + ")");
    }
    @Override
    public void onUpgrade(SQLiteDatabase sqLiteDatabase, int i, int i1) {
        sqLiteDatabase.execSQL("DROP TABLE IF EXISTS " + PERSON_TABLE_NAME);
        onCreate(sqLiteDatabase);
    }
}
```

如果你在一款移动应用开发过程中使用了 SQLite 作为存储机制，并且对数据的安全性具有强烈需求，那么你的开发环境需要一种特别的方法来处理这些问题。其中一种比较普遍接受的方法是使用 [SQLCipher](https://www.zetetic.net/sqlcipher) 方法来为数据库进行加密处理。SQLCipher 是 SQLite 的一种扩展，它为数据库文件提供了透明256位 AES 加密手段，这种手段使得开发一个安全的应用变得更简单，并且能更好地保护最终使用的用户的数据。使用 SQLCipher 的主要优点是，它提供了一个非常健壮的加密数据库的系统，而且它对不同的移动平台有着不同的版本支持，做到了良好的兼容性。不仅如此，针对不同等级的安全需求和服务支持，它提供了社区版（免费）和商用版两种版本以供选择。

### 数据同步性

用户在离线模式下所进行的一些操作，执行并与中心存储进行同步是一款支持离线的应用的核心功能之一。在很多情况下，你的应用可能会有客户端的存储以及服务端的存储，应用会处理数据在客户端和服务端之间的流动。通常来说，一旦应用中包括了离线功能的特性，你会碰到用户修改属于服务器一边控制的数据，或者，类似的，改动客户端方面所持有的数据这样的一些情况。这已经是众多可能性中最完整、最复杂的情景了。构建支持双向同步的离线应用是很吸引人的，这是目前为止所有方式中最为复杂的一个。在这种方式下，你的数据同步逻辑应当在同步过后确保数据最新，并在移动端和服务器端中保持一致。

要实现这个功能可以通过向客户端和服务器之间应该同步的每个对象添加一些“跟踪”字段来实现，反之亦然。这些字段大概长这样：‘**last_updated_on**’，'**created_on**'，如果数据没有在物理层面上移除就添加：‘**deleted_on**’ 字段。需要注意的是，从服务器数据库中删除了的行数据，所对应的在移动端保存的数据也需要删除。如果数据从服务器的硬盘中抹掉了，你的应用架构需要提供一种能保持已删数据信息的方法。将它转换成表结构来说，它可以是一张包括 ‘**table_name**’，‘**deleted_on**’，‘**server_record_id**’ 等字段的名为 '**deleted_record**' 的表。根据我以往的经验，我会推荐你在每一个表中都插入一列名为 '**last_updated_on**' 的时间戳字段来记录每次因为用户修改或者更新数据而进行同步的时间。接着，当客户端向服务端请求一次同步时，你的处理逻辑会迭代并检查所有表的 ‘**last_updated_on**’ 来判断本次同步的时间戳是否比最新记录更新。如果是，代码逻辑需要将该时间戳放进数据集合中，并且返回给客户端。在记录被物理抹掉的情况下，你必须读取 ‘**deleted_record**’ 表的所有新的行，并在客户端数据库中执行所提取的数据。在这种情况下如果数据两边都被修改了，那么你不能使用数据库的 ID 来作为数据对象的标记。因为可能在不同的设备创建的新对象中包含了相同的 ID。为了保证在不同设备上数据对象的唯一性，你需要使用一些独一无二的标识符。在我们的实现方式里，我们叫这个字段为 ‘**synchronization_key**’ 并且，一般它会包含 UUID 的内容。

## 总结

让我们在总结一下在此篇教程中所提到的所有内容：我们生活在一个技术飞速发展的时代，使用移动设备的用户不仅对应用的期望要求很高，而且需要他们的应用具有更加良好的性能和用户体验。如今，应用的离线支持已经变成了用户愿望清单中的一部分，而且不容忽视。通过运用各种各样的对于支持离线状态的方式方法，你将大大提高移动应用的用户体验，并且能够爆发团队的生产力。


## 译者评

初次翻译，必有多多问题，请包涵。

此篇文章涵盖的内容较广，但涉及每个的内容深度较浅。给一款应用提供相应的离线支持是一个比较复杂的命题：它所涉及的方式多种多样，所涉及需要保存的数据类型也是千奇百怪，设计一套高效的、有针对性的、数据安全的离线缓存架构，我认为是持久层、表现层和业务架构综合调配的产物。它不能独立于任何一个层面来运行，相反地，它可能会依赖其他的扩展来增强它的实现，例如加密库等。

***Fin***