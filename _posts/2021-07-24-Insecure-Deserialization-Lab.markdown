---
layout: post
title:  "Insecure deserialization in Java (PortswiggerLab)"
date:   2021-07-24 15:00:00 +0100
categories: writeup portswigger
---

### Introduction

A couple weeks ago I got a notification, informing me of [Michael Stepankin's](https://twitter.com/artsploit) new [research](https://portswigger.net/research/pre-auth-rce-in-forgerock-openam-cve-2021-35464) about an [insecure deserialization](https://portswigger.net/web-security/deserialization) in Java. It's been a while since I solved an old `PHP` deserialization challenge so I tought I had to review my knowledge about it. To do so, I decided to rush trough some of the `easy` and `practitioner` labs over at PortSwigger. I kept the lab for which I'm writing this writup right now for last. It's called [Developing a custom gadget chain for Java deserialization](https://portswigger.net/web-security/deserialization/exploiting/lab-deserialization-developing-a-custom-gadget-chain-for-java-deserialization) and it's marked as an `expert` lab. I will say, I had to peek into the solution for the last part of this challenge but without further ado let's look at it!

### Research
The lab description says:
>> This lab uses a serialization-based session mechanism. If you can construct a suitable gadget chain, you can exploit this lab's insecure deserialization to obtain the administrator's password. 
To solve the lab, gain access to the source code and use it to construct a gadget chain to obtain the administrator's password. Then, log in as the administrator and delete Carlos's account.
You can log in to your own account using the following credentials: wiener:peter

Let's setup our tools. I launched burp, opened the included browser and set the scope to the lab url. First thing we can do is loggin in with the account that is given to us. After that, any request to our target has a session cookie set:

![Normal request](/assets/Deserialization_normalrequest.png)

Inspecting the source with `CTRL+U` or looking at Burp Suite's `Site map`, shows a hidden path:

![Leaked Path](/assets/Deserialization_pathleak.png)

Turns out directory listing is active on the `/backup` endpoint and it exposes two files: `AccessTokenUser.java` and `ProductTemplate.java`. So probably java is used to generate our session cookie. Let's look at the files:

```java
//AccessTokenUser.java
package data.session.token;

import java.io.Serializable;

public class AccessTokenUser implements Serializable
{
    private final String username;
    private final String accessToken;

    public AccessTokenUser(String username, String accessToken)
    {
        this.username = username;
        this.accessToken = accessToken;
    }

    public String getUsername()
    {
        return username;
    }

    public String getAccessToken()
    {
        return accessToken;
  
```

```java
//ProductTemplate.java
package data.productcatalog;

import common.db.ConnectionBuilder;

import java.io.IOException;
import java.io.ObjectInputStream;
import java.io.Serializable;
import java.sql.Connection;
import java.sql.ResultSet;
import java.sql.SQLException;
import java.sql.Statement;

public class ProductTemplate implements Serializable
{
    static final long serialVersionUID = 1L;

    private final String id;
    private transient Product product;

    public ProductTemplate(String id)
    {
        this.id = id;
    }

    private void readObject(ObjectInputStream inputStream) throws IOException, ClassNotFoundException
    {
        inputStream.defaultReadObject();

        ConnectionBuilder connectionBuilder = ConnectionBuilder.from(
                "org.postgresql.Driver",
                "postgresql",
                "localhost",
                5432,
                "postgres",
                "postgres",
                "password"
        ).withAutoCommit();
        try
        {
            Connection connect = connectionBuilder.connect(30);
            String sql = String.format("SELECT * FROM products WHERE id = '%s' LIMIT 1", id);
            Statement statement = connect.createStatement();
            ResultSet resultSet = statement.executeQuery(sql);
            if (!resultSet.next())
            {
                return;
            }
            product = Product.from(resultSet);
        }
        catch (SQLException e)
        {
            throw new IOException(e);
        }
    }

    public String getId()
    {
        return id;
    }

    public Product getProduct()
    {
        return product;
    }
}
```

So the first one is a simple `Serializable` class with just two variables: `username` and `accessToken` . The second class has some logic in it. The readObject method is executed when an instance of the `ProductTemplate` class is deserialized. Overwriting this method gives the programmer the ability to control the deserialization process (and an attacker the ability to exploit this process!). There is some code and at the end the private variable `id` is used in an `SQL` query. Is this our entry point?

### Testing The Application

At this point we need a fast way to serialize and deserialize java objects. The best way to do this is writing a java program that does this for us. The lab provides a [generic Java program for serializing objects](https://github.com/PortSwigger/serialization-examples/tree/master/java/generic) . I used [jdoodle](https://www.jdoodle.com/online-java-compiler-ide/), an online IDE. Notice that the two leaked source code files are inside a package: `data.productcatalog` and `data.session.token`. To make things work we need to recreate this structure. Something like this:

![Directories](/assets/Deserialization_directories.png)

The AccessTokenUser.java file is exactly like the leaked one while for the ProductTemplate class I had to take out some logic because some files are missing (like the `Product` class). This isn't a problem however, because the serialization process gives the same result. This is the cleaned up `ProductTemplate.java`:

![ProductTemplate.java](/assets/Deserialization_producttemplate.png)

Now we just need to adapt the program that is given to us to our classes and we have a ready to use `serialize and deserialize` program. (I will show the complete code in the next section).

The ProductTemplate constructor accepts an `id`. Let's serialize this object with a random `id`. Now we can try to substitue our cookie with this new object and send the request. Remember to correctly `Base64` and `URL` encode things. Sending the request gives us a java error message! It says that It was expecting an AccessToken class but deserialized a ProductTemplate class and so the program crashes. For our purpouse however this good, at this point the readObject Method was already executed with our controlled `id` parameter!

### Exploiting the logic

We already have seen that the `id` is used inside an `sql query`. Let's try to send a serialized  `ProductTemplate` with an `id` set to `'`. It gives an SQL error! We can now change the payload and try different approaches. At first I used the `ORDER BY` [technique](https://portswigger.net/web-security/sql-injection/union-attacks#determining-the-number-of-columns-required-in-an-sql-injection-union-attack) to enumerate the number of columns and found out they where 8. At this point I changed approach and tryed error based Injection. 

```
--This gives error:
'; SELECT CASE WHEN (1=1) THEN cast(1/0 as text) ELSE 'a' END FROM users -- 
--This doesn't:
'; SELECT CASE WHEN (1=0) THEN cast(1/0 as text) ELSE 'a' END FROM users -- 

```

I guessed there was a table called `users` and in fact I was right (Otherwise we could have enumerate the tables). 

>>Now I had a problem that I still haven't solved. The above queries work only if the conditions inside the parenthesis are constant like `1=1` or `1=0` . Using something like `username = 'administrator' ` to test if the user administrator exists gives an error. The problem is that using `username = 'nonexistent'` gives an error too altrough the `nonexistent` user doesn't exist.  It almost seems like the value of the `TRUE` condition (`cast(1/0 ad text)`) is evaluated in any case if the `CONDITION` contains non constant values. 

Moving on, Timing based attack was possible:
```
--Server needs 10 seconds to respond:
'; SELECT CASE WHEN (1=1) THEN pg_sleep(10) ELSE 'a' END FROM users -- 
--Responds immediatly:
'; SELECT CASE WHEN (1=0) THEN pg_sleep(10) ELSE 'a' END FROM users -- 

```

We now could leak the administrator's password one character at a time. I already have solved challenges using this attack but in this case I felt I was going the wrong way. The server responded with the complete `error code` every time so we have a lot of additional informations. At this point I looked into the solution for a little hint. 

`'UNION SELECT null,null,null,cast(password as numeric),null,null,null,null FROM users WHERE username='administrator' ; -- `

Using something like the above gives an error because password is a text and can't be cast into a numeric value but the value of password is reflected in the response! Here's the code I used to generate The correct serialized object:

![Final](/assets/Deserialization_final.png)

### The end
Now that we leaked the admin's password, we just need to log into his account, access the admin panel and delete `carlos` user.