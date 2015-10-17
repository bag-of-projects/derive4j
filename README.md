# derive4j
[ ![Download](https://api.bintray.com/packages/derive4j/derive4j/derive4j/images/download.svg) ](https://bintray.com/derive4j/derive4j/derive4j/_latestVersion)

Java 8 annotation processor and framework for deriving algebraic data types constructors, morphisms. Optics and typeclasses comming soon.

This project has a special dedicace to Tony Morris' blog post [Debut with a catamorphism] (http://blog.tmorris.net/posts/debut-with-a-catamorphism/index.html)

So what can this project do for us, poor functional programmer stuck with a legacy language called Java?
A good deal of what is available for free in better languages like haskell: pattern matching, laziness...
An example being worth a thousand words:

```java
package org.derive4j.exemple;

import fj.*;
import fj.data.Option;
import fj.data.optic.*;
import org.derive4j.*;

import static fj.P.p;
import static fj.data.Option.none;
import static fj.data.Option.some;
import static fj.data.optic.Lens.lens;
import static fj.data.optic.Optional.optional;
import static fj.data.optic.Prism.prism;

/**
 * A data type to modelize an http request. Abstract because concrete
 * implementation will be generated, by Derive4J (annotation processor
 * for the @Data annotation). Default @Data flavour is JDK,
 * here we specify FJ (Functional Java), also available is Fugue and Fugue2.
 * The flavour is used to determine which implementation of 'Option' or
 * 'Function' will be used by generated code.
 */
@Data(flavour = Flavour.FJ)
public abstract class Request {

  /**
   * First we start by defining a 'visitor' for our datatype:
   */

  // the 'visitor' interface:
  interface Cases<X> {
    X GET(String path);
    X DELETE(String path);
    X PUT(String path, String body);
    X POST(String path, String body);
  }

  // the 'accept' method:
  public abstract <X> X match(Cases<X> cases);

  /**
   * Alternatively, if you prefer a more FP style, you can define a
   * catamorphism instead (equivalent to the visitor above, most useful
   * for standard data type like Option, Either, List...):
   */
  public abstract <X> X match(@FieldNames("path") F<String, X> GET,
                              @FieldNames("path") F<String, X> DELETE,
                              @FieldNames({"path", "body"}) F2<String, String, X> PUT,
                              @FieldNames({"path", "body"}) F2<String, String, X> POST);

  /**
   * Derive4J philosophy is to be as safe and consistent as possible.
   * That is why Object.{equals, hashCode, toString} are not implemented
   * by generated classes by default. Nonetheless, as a concession to legacy,
   * it is possible to force Derive4J to implement them, by declaring them abstract:
   */
  @Override
  public abstract int hashCode();
  @Override
  public abstract boolean equals(Object obj);
  @Override
  public abstract String toString();

  /**
   * Now run compilation and a 'Requests' classe will be generated, by default with
   * the same visibility as 'Request'.
   * If You want you can specify the visibility Package in the @Data annotation
   * and expose only public methods here, and delegate to the generated Requests class.
   * eg. for constructors:
   */
  public static Request GET(String path) {
    return Requests.GET(path);
  }
  public static Request PUT(String path, String body) {
    return Requests.PUT(path, body);
  }

  /**
   * Derive4J provides first class support for lazy construction of data types:
   */
  public static Request lazy(F0<Request> requestExpression) {
    // the requestExpression will be lazy-evaluated on the first call
    // to the 'match' method of the returned instance:
    return Requests.lazy(requestExpression);
  }

  /**
   * Then you can enrich your class with whatever methods you like,
   * using generated static methods to trivialize your implementation:
   */

  /**
   * OOP style 'getters':
   */

  public final String path() {
    return Requests.getPath(this);
  }

  public final Option<String> body() {
    return Requests.getBody(this);
  }

  /**
   * OOP style 'withers':
   */

  public final Request withPath(String newPath) {
    return Requests.setPath(newPath).f(this);
  }

  public final Request withBody(String newBody) {
    // if there is no body field (eg. GET, DELETE) then the original request
    // is returned (no modification):
    return Requests.setBody(newBody).f(this);
  }

  /**
   * FP style Optics (in the future, will be generated by a derive4j 'plugin'):
   */

  /**
   * Lenses: optics focused on a field present for all datatype contructors
   * (getter cannot 'failed'):
   */
  public static final Lens<Request, String> _path = lens(
      Requests::getPath,
      Requests::setPath);
  // which is Equivalent to:
  public static final Lens<Request, String> _pathPatternMatch = lens(
      // getter function:
      Requests.match()
          .GET(path -> path)
          .DELETE(path -> path)
          .PUT((path, body) -> path)
          .POST((path, body) -> path),
      // setter function:
      newPath ->
          Requests.match()
              .GET(path -> Requests.GET(newPath))
              .DELETE(path -> Requests.DELETE(newPath))
              .PUT((path, body) -> Requests.PUT(newPath, body))
              .POST((path, body) -> Requests.POST(newPath, body))
  );

  /**
   * Optional: optics focused on a field that may not be present for all contructors
   * (getter return an 'Option'):
   */
  public static final Optional<Request, String> _body = optional(
      Requests::getBody,
      Requests::setBody);
  // Equivalent to:
  public static final Optional<Request, String> _bodyPatternMatch = optional(
      // getter function:
      Requests.match()
          .PUT((path, body) -> some(body))
          .POST((path, body) -> some(body))
          .otherwise(() -> none()),
      // setter function:
      newBody ->
          Requests.match()
              .GET(path -> Requests.GET(path)) // or with method reference:
              .DELETE(Requests::DELETE)
              .PUT((path, body) -> Requests.PUT(path, newBody))
              .POST((path, body) -> Requests.POST(path, newBody))
  );



  /**
   * Prism: optics focused on a specific constructor:
   */
  public static final Prism<Request, String> _GET = prism(
      // Getter function
      Requests.match()
          .GET(fj.data.Option::some)
          .otherwise(Option::none),
      // Reverse Get function (aka constructor)
      Requests::GET);

  // If there more than one field, we use a tuple as the prism target:
  public static final Prism<Request, P2<String, String>> _POST = prism(
      // Getter:
      Requests.match()
          .POST((path, body) -> some(p(path, body)))
          .otherwise(Option::none),
      // reverse get (construct a POST request given a P2<String, String>):
      p2 -> Requests.POST(p2._1(), p2._2()));
}

```
# Use it in your project
Derive4J is available via Jcenter.
## Maven:
```xml
<repositories>
  <repository>
    <id>bintray</id>
    <url>http://jcenter.bintray.com</url>
  </repository>
</repositories>
...
<dependency>
  <groupId>org.derive4j</groupId>
  <artifactId>derive4j</artifactId>
  <version>0.3</version>
</dependency>
```
##Gradle
```
compile(group: 'org.derive4j', name: 'derive4j', version: '0.3', ext: 'jar')
```
