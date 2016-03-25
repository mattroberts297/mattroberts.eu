---
layout: post
title:  "Publishing from SBT to Maven Central"
date:   2014-02-05 19:30:00
summary: "Publish an artifact using the sbt-pgp plugin and the Sonatype Nexus Repository Manager."
tags:
- scala
- sbt
- open source
---

This is how I published [SLF4S][slf4s] to Maven Central.
More information can be found at the [SBT docs][sbt-docs] and the [Sonatype docs][sonatype-docs].
The steps I followed were:

- [sign up][sonatype-signup] for an account at Sonatype;
- [create a ticket][sonatype-ticket] at Sonatype;
- create a key pair;
- publish the public key;
- configure SBT;
- configure project;
- publish to Sonatype; and
- promote from Sonatype to Maven Central.

### Create a key pair
{% highlight bash %}
# Install gpg (mac specific)
$ sudo port install gnupg

# Generate a key pair
$ gpg --gen-key

# List public keys
$ gpg --list-keys
/Users/Matt/.gnupg/pubring.gpg
------------------------------
pub   2048R/E547E5E3 2014-02-06
uid                  Matt Roberts <mattroberts@gmail.com>
sub   2048R/CC09511A 2014-02-06
{% endhighlight %}

### Publish the public key
{% highlight bash %}
# Publish the public key
$ gpg --keyserver hkp://pool.sks-keyservers.net --send-keys E547E5E3

# Export your private key (and put it somewhere safe)
$ gpg --export-secret-key -a 'Matt Roberts' > private.key
{% endhighlight %}

### Configure SBT
##### .sbt/0.13/sonatype.sbt
{% highlight scala %}
credentials += Credentials("Sonatype Nexus Repository Manager",
                           "oss.sonatype.org",
                           "your_sonatype_username",
                           "your_sonatype_password")
{% endhighlight %}

##### .sbt/0.13/plugins/gpg.sbt
{% highlight scala %}
addSbtPlugin("com.typesafe.sbt" % "sbt-pgp" % "0.8")
{% endhighlight %}

### Configure project
##### project/Build.scala
{% highlight scala %}
import sbt._
import Keys._

object Slf4sBuild extends Build {
  lazy val slf4s = Project("sl4s-api", file(".")) settings(
    organization := "org.slf4s",
    name := "slf4s-api",
    scalaVersion := "2.10.3",
    version := "1.7.5",
    publishMavenStyle := true,
    publishArtifact in Test := false,
    pomIncludeRepository := { _ => false },
    licenses := Seq("MIT" -> url("http://opensource.org/licenses/MIT")),
    homepage := Some(url("http://slf4s.org/")),
    scmInfo := Some(ScmInfo(
      url("https://github.com/mattroberts297/slf4s"),
      "https://github.com/mattroberts297/slf4s",
      None)),
    pomExtra := (
      <developers>
        <developer>
          <id>mattroberts297</id>
          <name>Matt Roberts</name>
          <email>mattroberts297@gmail.com</email>
          <url>http://mattro.be/rts/</url>
        </developer>
      </developers>
    ),
    publishTo := {
      val nexus = "https://oss.sonatype.org/"
      if (isSnapshot.value)
        Some("snapshots" at nexus + "content/repositories/snapshots")
      else
       Some("releases"  at nexus + "service/local/staging/deploy/maven2")
    },
    libraryDependencies ++= Seq(
      "org.slf4j" % "slf4j-api" % "1.7.5",
      "org.scalatest" %% "scalatest" % "2.0" % "test",
      "org.mockito" % "mockito-all" % "1.9.5" % "test"
    )
  )
}
{% endhighlight %}

### Publish to Sonatype
{% highlight bash %}
# Open SBT
$ sbt

# Publish to Sonatype
> publish-signed
{% endhighlight %}

### Promote from Sonatype to Maven Central

Head over to [Sonatype OSS][sonatype-oss] and you should see your artifact in either the snapshots repository or a staging repository.

You have a few options in the staging repository:
- Close (Sonatype will check your release meets Maven Central's requirements)
- Drop (if you fail to meet Maven Central's requirements)
- Release (if you met the requirements)

[slf4s]: http://slf4s.org
[sbt-docs]: http://www.scala-sbt.org/release/docs/Community/Using-Sonatype.html
[sonatype-docs]: https://docs.sonatype.org/display/Repository/Sonatype+OSS+Maven+Repository+Usage+Guide#SonatypeOSSMavenRepositoryUsageGuide-2
[sonatype-signup]: https://docs.sonatype.org/display/Repository/Sonatype+OSS+Maven+Repository+Usage+Guide#SonatypeOSSMavenRepositoryUsageGuide-2.Signup
[sonatype-ticket]: https://docs.sonatype.org/display/Repository/Sonatype+OSS+Maven+Repository+Usage+Guide#SonatypeOSSMavenRepositoryUsageGuide-3.CreateaJIRAticket
[sonatype-oss]: https://oss.sonatype.org/
