# slick-joda-mapper

[![Build Status](https://travis-ci.org/tototoshi/slick-joda-mapper.png)](https://travis-ci.org/tototoshi/slick-joda-mapper)

Enables you to use joda-time with Slick.
You can persist `DateTime`, `Instant`, `LocalDateTime`, `LocalDate`, `LocalTime`, `DateTimeZone` with Slick.

# Usage


## For Slick 2.x

```scala
libraryDependencies ++= Seq(
    "com.typesafe.slick" %% "slick" % "2.1.0",
    "joda-time" % "joda-time" % "2.4",
    "org.joda" % "joda-convert" % "1.6",
    "com.github.tototoshi" %% "slick-joda-mapper" % "1.2.0"
)
```

Import xJodaSupport(H2JodaSupport, PostgresJodaSupport, MySQLJodaSupport...) class suitable for the database you use.
For example, import `H2JodaSupport` if you are using H2Driver.

```scala
import scala.slick.driver.H2Driver.simple._
import com.github.tototoshi.slick.H2JodaSupport._
```

Different drivers __can't__ be mixed, You __can't__ do the following.

```scala
import scala.slick.driver.H2Driver.simple._
import com.github.tototoshi.slick.JdbcJodaSupport._
```

Write your own `JdbcSupport` when you want to write db agnostic model class.

```scala
object PortableJodaSupport extends com.github.tototoshi.slick.GenericJodaSupport(yourAbstractDriver)

// with play-slick
object PortableJodaSupport extends com.github.tototoshi.slick.GenericJodaSupport(play.api.db.slick.Config.driver)

import PortableJodaSupport._
```

### Code generation

Write a custom code generator that replaces `java.sql.Timestamp` with whatever Joda classes you prefer.  

This example maps `java.sql.Timestamp` to `org.joda.time.DateTime` using the Postgres support.  When you modify it to suit your needs, make sure that the imports refer to the correct `JdbcSupport` class for your database and Joda classes.

```scala
import scala.slick.{model => m}
import scala.slick.codegen.SourceCodeGenerator

class CustomSourceCodeGenerator(model: m.Model) extends SourceCodeGenerator(model) {

  // add some custom imports
  // TODO: fix these imports to refer to your JdbcSupport and your Joda imports
  override def code = "import com.github.tototoshi.slick.PostgresJodaSupport._\n" + "import org.joda.time.DateTime\n" + super.code

  override def Table = new Table(_) {
    override def Column = new Column(_) {

      // munge rawType -> SQL column type HERE (scaladoc in Slick 2.1.0 is outdated or incorrect, GeneratorHelpers#mapJdbcTypeString does not exist)
      // you can filter on model.name for the column name or model.tpe for the column type
      // your IDE won't like the String here but don't worry, the return type the compiler expects here is String
      override def rawType = model.tpe match {
        case "java.sql.Timestamp"               => "DateTime" // kill j.s.Timestamp
        case _ => {
//          println(s"${model.table.table}#${model.name} tpe=${model.tpe} rawType=${super.rawType}")
          super.rawType
        }
      }
    }
  }
}
```
Then write a simple app harness to run code generation:

```scala

import scala.slick.driver.JdbcProfile

object CodeGen extends App {

  // http://slick.typesafe.com/doc/2.1.0/code-generation.html

  val slickDriver = "scala.slick.driver.PostgresDriver"  // TODO: replace this with your Slick driver
  val jdbcDriver = "org.postgresql.Driver"               // TODO: replace this with your JDBC driver
  val url = "jdbc:postgresql://127.0.0.1:5432/foo"       // TODO: replace this with your database's JDBC URL
  val outputFolder = "src/main/scala"                    // TODO: or whatever output folder you're in the mood for
  val pkg = "foo"                                        // TODO: your package name
  val user = "postgres"                                  // TODO: database username - optional, use forURL supports both with and without credentials
  val password = ""                                      // TODO: database password - optional, use forURL supports both with and without credentials

  val driver: JdbcProfile = scala.slick.driver.PostgresDriver  // TODO: replace this with your Slick driver

  val db = {
    // UNCOMMENT this if your database doesn't need credentials
    // driver.simple.Database.forURL(url, jdbcDriver)
    driver.simple.Database.forURL(url, driver = jdbcDriver, user = user, password = password)
  }

  db.withSession { implicit session =>
    new CustomSourceCodeGenerator(driver.createModel()).writeToFile(slickDriver, outputFolder, pkg)
  }

}

```

You can run and keep adjusting your source code generation using `sbt run`.  If you get compile errors on the generated `Tables.scala`, just delete it and try again.

## For Slick 1.x

```scala
libraryDependencies += "com.github.tototoshi" %% "slick-joda-mapper" % "0.4.1"
```

```scala
import com.github.tototoshi.slick.JodaSupport._
```

# Example


https://github.com/tototoshi/slick-joda-mapper/blob/master/src/test/scala/com/github/tototoshi/slick/JodaSupportSpec.scala


# Changelog

## 1.2.0
 - Support Slick 2.1.0.

## 1.1.0
 - Added DateTimeZone support.

## 1.0.1
 - Added JdbcJodaSupport.
