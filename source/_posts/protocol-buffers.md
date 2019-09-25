---
title: protocol buffers
date: 2019-01-13 03:54:44
tags: 
- google
- protocol-buffers
categories:
- tools
---



本教程旨在让我们学会：

1. 定义一个.proto文件
2. 使用 protocol buffer complier
3. 用户 Java protocol buffer API 来读写信息



#### 首先我们结合实际的应用场景来了解一下，为什么需要protocol buffers?

这里我们是用一个通讯录对文件的读写来完成对用户通讯信息的读写的例子作为介绍。

当我们要序列化和以及恢复这种有结构的数据，有如下几种方法。

- 利用Java的序列化机制。但是存在如下问题：
  - 存在许多众所周知的问题 (see Effective Java, by Josh Bloch pp. 213)
  - 不能很好地和其他语言C++/Python共享数据（和语言的耦合性太高）
- 自定义一个特定的（ad-hoc）方式把数据项编码成一个字符串。比如将四个整数编码为"12:3:-23:67"。这是一种简单而灵活的方法，尽管它需要编写一次性（这里的英文是one-off, 意思为happen only once，我理解是只适用于这个场景的解码和编码方法，因此翻译为一次性的）的编码和解码的代码，而且解码会产生很小的运行时间成本。所以这种方式很适合编码简单的数据。
- 将数据序列化为XML。 这种方法可能非常有吸引力，因为XML是可读的，并且有许多语言的绑定库。 如果您想与其他应用程序/项目共享数据，这可能是一个不错的选择。 然而，XML是众所周知的空间密集型，<u>编码/解码可能会给应用程序带来巨大的性能损失</u>。 另外，导航XML DOM树比导航类中的简单字段通常要复杂得多。

Protocol buffers是解决这个问题的灵活，高效的自动化解决方案。

你只需要写一个你想要存储的数据结构的描述.proto文件。之后protocol buffer compiler会创建一个类。这个类

1. 用高效的二进制格式实现了自动编码和解码。
2. 对字段提供了getters和setters。
3. 将读写的细节作为一个单元处理
4. 支持拓展格式，并且保持对旧的编码的数据读取的兼容。



#### 下面介绍具体的使用：

- 定义.proto文件的方式

  - Extend info
    - package（区分不同project）
    - java_package（java的包, default=package）
    - java_outer_classname（java的类名, default=filename2camelcase）
  - Massage declaration
    - field types(default value): bool(false), int32(0), float(0), double(0), string("")
    - enum定义
    - tag: field在二进制编码中使用的标记。所以1-15（需要少于一个字节）一般赋予常用的和repeated的元素。>=16的给一些不常使用的optional元素。
    - modifiers: required(少用，对拓展限制很强), optional, repeated

  ```protobuf
  syntax = "proto2";
  
  package tutorial;
  
  option java_package = "com.example.tutorial";
  option java_outer_classname = "AddressBookProtos";
  
  message Person {
    required string name = 1;
    required int32 id = 2;
    optional string email = 3;
  
    enum PhoneType {
      MOBILE = 0;
      HOME = 1;
      WORK = 2;
    }
  
    message PhoneNumber {
      required string number = 1;
      optional PhoneType type = 2 [default = HOME];
    }
  
    repeated PhoneNumber phones = 4;
  }
  
  message AddressBook {
    repeated Person people = 1;
  }
  ```

- 编译该文件(-I用于import)

  ```
  protoc -I=$SRC_DIR --java_out=$DST_DIR $SRC_DIR/addressbook.proto
  ```

- 生成的Class文件如何使用（生成了什么，提供了什么功能）

  - 生成了Message对应的class。每个class有自己的Builder。且由protocol compiler生成的类都是对应JAVA风格的驼峰命名。<u>这里建议在.proto文件中使用小写字母+下划线</u>。（因为protocol compiler是在小写字母+下划线规范上，解析生成对应语言的风格的代码）

  - Message: 信息对象，immutable，不可被修改

    - Singular field: has, get
    - Repeated fields: getList, getCount, get(index)
    - 标准方法，Message接口声明的
      - isInitialized(): checks if all the required fields have been set.
      - toString(): returns a human-readable representation of the message, particularly useful for debugging
    - 序列化和解码
      - byte[] toByteArray();: serializes the message and returns a byte array containing its raw bytes.
      - static Person parseFrom(byte[] data);: parses a message from the given byte array.
      - void writeTo(OutputStream output);: serializes the message and writes it to an OutputStream.
      - static Person parseFrom(InputStream input);: reads and parses a message from an InputStream.

  - Builder: 信息对象的构造器，用来创建massage对象

    - Singular field: has, get, set, clear
    - Repeated fields: getList, getCount, get(index), set(index, value), add(value), addAll(value), clear
    - 标准方法，Message.Builder接口声明的
      - isInitialized()
      - toString()
      - mergeFrom(Message other):  merges the contents of other into this message, overwriting singular scalar fields, merging composite fields, and concatenating repeated fields
      - clear(): clears all the fields back to the empty state

  - Enums and Nested Classes:

    枚举类型会自动在声明的类的内部生成

  > 由于协议缓冲区类基本上是哑数据持有者（如C中的strcuts），所以注意这里要拓展数据类的功能，比如添加一些行为，不可以通过继承的方式，因为这将打破protocol buffer内部的机制。可以通过包装(wrap)数据类到特定于应用程序的类中。（适配器模式）

- 尝试写入文件

```java
import com.example.tutorial.AddressBookProtos.AddressBook;
import com.example.tutorial.AddressBookProtos.Person;
import java.io.BufferedReader;
import java.io.FileInputStream;
import java.io.FileNotFoundException;
import java.io.FileOutputStream;
import java.io.InputStreamReader;
import java.io.IOException;
import java.io.PrintStream;

class AddPerson {
  // This function fills in a Person message based on user input.
  static Person PromptForAddress(BufferedReader stdin,
                                 PrintStream stdout) throws IOException {
    Person.Builder person = Person.newBuilder();

    stdout.print("Enter person ID: ");
    person.setId(Integer.valueOf(stdin.readLine()));

    stdout.print("Enter name: ");
    person.setName(stdin.readLine());

    stdout.print("Enter email address (blank for none): ");
    String email = stdin.readLine();
    if (email.length() > 0) {
      person.setEmail(email);
    }

    while (true) {
      stdout.print("Enter a phone number (or leave blank to finish): ");
      String number = stdin.readLine();
      if (number.length() == 0) {
        break;
      }

      Person.PhoneNumber.Builder phoneNumber =
        Person.PhoneNumber.newBuilder().setNumber(number);

      stdout.print("Is this a mobile, home, or work phone? ");
      String type = stdin.readLine();
      if (type.equals("mobile")) {
        phoneNumber.setType(Person.PhoneType.MOBILE);
      } else if (type.equals("home")) {
        phoneNumber.setType(Person.PhoneType.HOME);
      } else if (type.equals("work")) {
        phoneNumber.setType(Person.PhoneType.WORK);
      } else {
        stdout.println("Unknown phone type.  Using default.");
      }

      person.addPhones(phoneNumber);
    }

    return person.build();
  }

  // Main function:  Reads the entire address book from a file,
  //   adds one person based on user input, then writes it back out to the same
  //   file.
  public static void main(String[] args) throws Exception {
    if (args.length != 1) {
      System.err.println("Usage:  AddPerson ADDRESS_BOOK_FILE");
      System.exit(-1);
    }

    AddressBook.Builder addressBook = AddressBook.newBuilder();

    // Read the existing address book.
    try {
      addressBook.mergeFrom(new FileInputStream(args[0]));
    } catch (FileNotFoundException e) {
      System.out.println(args[0] + ": File not found.  Creating a new file.");
    }

    // Add an address.
    addressBook.addPeople(
      PromptForAddress(new BufferedReader(new InputStreamReader(System.in)),
                       System.out));

    // Write the new address book back to disk.
    FileOutputStream output = new FileOutputStream(args[0]);
    addressBook.build().writeTo(output);
    output.close();
  }
}
```

- 尝试读取文件

```java
import com.example.tutorial.AddressBookProtos.AddressBook;
import com.example.tutorial.AddressBookProtos.Person;
import java.io.FileInputStream;
import java.io.IOException;
import java.io.PrintStream;

class ListPeople {
  // Iterates though all people in the AddressBook and prints info about them.
  static void Print(AddressBook addressBook) {
    for (Person person: addressBook.getPeopleList()) {
      System.out.println("Person ID: " + person.getId());
      System.out.println("  Name: " + person.getName());
      if (person.hasEmail()) {
        System.out.println("  E-mail address: " + person.getEmail());
      }

      for (Person.PhoneNumber phoneNumber : person.getPhonesList()) {
        switch (phoneNumber.getType()) {
          case MOBILE:
            System.out.print("  Mobile phone #: ");
            break;
          case HOME:
            System.out.print("  Home phone #: ");
            break;
          case WORK:
            System.out.print("  Work phone #: ");
            break;
        }
        System.out.println(phoneNumber.getNumber());
      }
    }
  }

  // Main function:  Reads the entire address book from a file and prints all
  //   the information inside.
  public static void main(String[] args) throws Exception {
    if (args.length != 1) {
      System.err.println("Usage:  ListPeople ADDRESS_BOOK_FILE");
      System.exit(-1);
    }

    // Read the existing address book.
    AddressBook addressBook =
      AddressBook.parseFrom(new FileInputStream(args[0]));

    Print(addressBook);
  }
}
```

- 如何拓展一个protocol buffer（新buffers向下兼容，旧buffers向上兼容）(<u>**考虑的是新旧代码上的兼容**</u>)

  - you *must not* change the tag numbers of any existing fields.
  - you *must not* add or delete any required fields.
  - you *may* delete optional or repeated fields.
  - you *may* add new optional or repeated fields but you must use fresh tag numbers (i.e. tag numbers that were never used in this protocol buffer, not even by deleted fields).

  对于删除的可选字段

  - 读取新的文件时，旧代码设置为默认值，如果没有默认值，则是对应类型的默认值。重复字段为空。
  - 读取旧的文件时，新代码会透明读取，即忽略被删除的字段。

  对于新增的可选字段

  - 读取新的文件时，旧代码会透明读取，即忽略被删除的字段。
  - 读取旧的文件时，新代码设置为默认值，如果没有默认值，则是对应类型的默认值。重复字段为空。

- 高级应用（反射）【以下为google翻译】

  协议消息类提供的一个关键特性是反射。 您可以迭代消息的字段并操作它们的值，而无需针对任何特定消息类型编写代码。 使用反射的一个非常有用的方法是将协议消息转换为其他编码（例如XML或JSON）以及从其他编码转换。 更高级的反射使用可能是找到两个相同类型的消息之间的差异，或者开发一种“协议消息的正则表达式”，在其中可以编写匹配特定消息内容的表达式。 如果您使用自己的想象力，可以将Protocol Buffers应用于比您最初期望的更广泛的问题！

#### Summary

本质上就是提供了一个中间层，完成数据的定义和数据在具体代码中实现（提供的API：getset，序列化，解码编码，builder-merge等）的解耦。



#### Reference

[protocol buffers java tutorial](https://developers.google.com/protocol-buffers/docs/javatutorial)