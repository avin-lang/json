# json

JSON for Avin: `Json` values, reading text with the line and column of any error, writing compact,
indented or ASCII text, the accessors that take a value apart, a streaming reader that reads a
document of any size as it arrives, and encoders and decoders derived for a program's own types.

The package's module is **`dev.avinlang.json`**, written in Avin; the import binds `json`. A program
depends on it in its `avin.hocon`:

```hocon
dependencies {
  json { git = "github.com/avin-lang/json", version = "0.0.3" }
}
```

or with `avin add github.com/avin-lang/json@0.0.1`. The package needs nothing beyond the standard
library.

```avin
import dev.avinlang.json
import dev.avinlang.json.Json

main()
  val event = json.parse("{\"id\": 7, \"cmd\": \"make\", \"tags\": [\"a\", \"b\"]}") match
    Ok(j) -> j
    Err(e) -> fail(s"bad event: $e")
  println(event("cmd").asString)                    // Some(make)
  println(event.get("exit").flatMap(x -> x.asInt))  // None
  println(event.with("exit", Json.num(0)))            // {"id":7,"cmd":"make","tags":["a","b"],"exit":0}
```

## Values

```text
data Json
  Null
  Bool(value: bool)
  Long(value: long)
  Real(value: real)
  Str(value: string)
  Arr(items: Vector[Json])
  Obj(members: Vector[(string, Json)])
```

- **A number is a `Long` when the text writes a whole number that fits a `long`**: digits with an
  optional `-` and neither a fraction nor an exponent, from `-9223372036854775808` to
  `9223372036854775807`. An identifier or a count is therefore exact however large. `-0` is the
  `Long` 0.
- **Every other number is a `Real`**: one with a fraction or an exponent (`1.0`, `1e3`, `-0.0`, which
  keeps its sign), and a whole number beyond a `long`, rounded to the nearest `real`. A number too
  large to be a finite `real` (`1e400`) is an error; one too small is 0 with its sign.
- **An object keeps every member in the order written**, a key written twice included, so a
  document read and written again comes back in its own order. `get` and `apply` answer the last
  member of a key written more than once.
- **Values compare by structure**: an object's members in order, so `{"a":1,"b":2}` and
  `{"b":2,"a":1}` are different values, and a `Long` is never equal to a `Real`.
- **A value displays as its compact text**, by `Display` and by `Debug`, so `println(j)` and
  `s"$j"` write it as JSON.

The companion makes values without spelling the variants: `Json.obj(("id", Json.num(7)))` is an
object of the members given, `Json.arr(items*)` an array, `Json.str(s)` a string and `Json.num(n)`
a `Long` for an `int` or a `long` and a `Real` for a `real`.

## Reading

```text
parse(text: string, maxDepth: int = defaultMaxDepth) -> Result[Json, JsonError]
parse(bytes: Array[byte], maxDepth: int = defaultMaxDepth) -> Result[Json, JsonError]

data JsonError(message: string, line: long, column: long, offset: long)
val defaultMaxDepth: int = 512
```

`parse` reads one value with nothing but white space around it, and answers the first thing wrong
with the text otherwise. The grammar is JSON's own and nothing more:

- **White space is space, tab, line feed and carriage return.** A form feed, a byte order mark or
  a no-break space is not white space, and there are no comments.
- **No trailing comma**, in an array or an object; an object's key is a string, followed by `:`.
- **A number has no leading zero, `+`, bare `.` or hexadecimal form**, and `NaN` and `Infinity` are
  not numbers. The words are `true`, `false` and `null`, whole and in lower case.
- **A string holds no control character below U+0020 unescaped.** Its escapes are `\"`, `\\`, `\/`,
  `\b`, `\f`, `\n`, `\r`, `\t` and `\u` with four hexadecimal digits in either case. A character
  beyond U+FFFF is a high surrogate escape followed at once by a low one, which read as the one
  character; a lone surrogate, high or low, is an error, since a `string` holds only characters.
  `\u0000` is the character U+0000.
- **Arrays and objects nest at most `maxDepth` deep**, so a hostile document cannot exhaust the
  stack: `[[1]]` is two deep, and a number, string or word alone is 0 deep. The limit is 512 unless
  a call gives another; the error is at the bracket that goes one deeper.
- **Bytes are read as UTF-8**; bytes that are not UTF-8 are an error at the first of them.

An error's `line` and `column` count from 1, a line ending at a line feed and a column counting
characters rather than bytes; `offset` is the byte offset in the text, counted from 0. All three are
`long`s, so a [stream](#reading-a-stream) of many gibibytes, on one line or many, is placed exactly.
It displays as its message and its place:

```text
unexpected ']' after ',' in an array at line 1, column 4
a high surrogate \uD800 with no low surrogate after it at line 3, column 12
a number with a leading zero at line 1, column 2
arrays and objects nested more than 512 deep at line 1, column 513
```

A string with no escape is one slice of the text, and the reading goes byte by byte with no
backtracking: a mebibyte document is read in tens of milliseconds.

## Reading a stream

A `JsonReader` reads JSON text as it arrives, from any `avin.io.Reader` (an
open file, a TCP or Unix connection, or a source the program writes), and gives it back one event
at a time without building the whole value. It holds the piece of text it is working through
(64 KiB at most from each read), the token it is reading and one entry for each array or object it
is inside, and never calls itself per level, so a document larger than memory, or nested as deep as
its limit allows, reads in a bounded amount of it: a hundred megabytes of records read in about
twelve mebibytes of resident memory, the program included.

```avin
import dev.avinlang.json
import dev.avinlang.json.JsonReader
import avin.fs

// The sum of every record's "bytes", read from a log far larger than memory.
totalBytes(path: string) -> Result[long, json.JsonError]
  val r = JsonReader.fromReader(fs.open(path).ok.get)
  var sum: long = 0
  r.beginArray()?
  while r.nextElement()?
    r.beginObject()?
    loop
      r.nextKey()? match
        None -> break
        Some("bytes") -> sum += r.readLong()?
        Some(_) -> r.skipValue()?
  r.finish()?
  Ok(sum)
```

```text
data JsonEvent
  StartObject
  EndObject
  StartArray
  EndArray
  MemberName(name: string)
  StringValue(value: string)
  LongValue(value: long)
  RealValue(value: real)
  BoolValue(value: bool)
  NullValue
  EndOfText
```

| Made by | Reads |
| --- | --- |
| `JsonReader.fromReader(source: Reader, maxDepth = defaultMaxDepth)` | The text `source` gives, a piece at a time as events are asked for |
| `JsonReader.fromString(text, maxDepth = defaultMaxDepth)` | `text` |
| `JsonReader.fromBytes(bytes, maxDepth = defaultMaxDepth)` | Text given as its UTF-8 bytes |

| Call | Gives |
| --- | --- |
| `next()` | The next event, moving past it |
| `peek()` | The next event, which `next()` then gives again |
| `depth()` | How many arrays and objects the reader is inside |
| `beginArray()`, `beginObject()` | Moves into the array or object that must come next |
| `nextElement()` | Whether the array has another element; at its end, moves past it and answers `false` |
| `nextKey()` | The next member's name, or `None` at the object's end, moving past it |
| `readString()`, `readBool()` | The next value, which must be of that kind |
| `readLong()`, `readInt()`, `readReal()` | The next value, a number, taken as `asLong`, `asInt` and `asReal` take one |
| `takeNull()` | Whether the next value is `null`, moving past it when it is; anything else is left for the next call |
| `skipValue()` | Moves past the next value, whatever it holds |
| `readValue()` | The next value whole, as a `Json` |
| `readDocument()` | The text's one value whole, and the end of the text after it |
| `finish()` | The end of the text, with only white space after the value |
| `error(message)` | A `JsonError` saying `message`, placed where the last event given or peeked began |

Each call but `depth`, `takeNull` and `error` answers a `Result`, so a decoder chains them with `?`.

- **The grammar is `parse`'s, and so are its faults**: every text `parse` refuses, a stream refuses
  with the same message, line, column and offset, however its source cuts it into pieces. A
  character split between two reads, a surrogate pair's two escapes split anywhere, and a number or
  a word cut in two are read whole. UTF-8 is checked as it is read, so bytes that are not UTF-8 are
  an error at the first of them, as in `parse`; where a grammar fault comes before them in the text,
  the stream, which has not read them yet, reports the grammar fault.
- **One value, then the end of the text.** After the value the reader gives `EndOfText`, and gives
  it again on every later call; anything but white space after the value is the fault `unexpected
  'x' after the value`.
- **A number is a `LongValue` or a `RealValue` by `parse`'s rule**: a whole number that fits a
  `long` is a `LongValue`, any other a `RealValue`. `readInt` and `readLong` take a whole `RealValue`
  within range (`3.0`, `1e3`) as `asInt` and `asLong` do, and refuse one that is not, naming it.
- **A value of the wrong kind** is an error at the event that was found, naming what was expected
  and what was found: `expected a string, found a number at line 2, column 9`.
- **A fault ends the reading.** The first fault in the text, or a failure of the source to read
  (`the text could not be read: timed out`, placed where the reading stopped), is the `Err` of the
  call that met it and of every call after it. A token the failure cut short is not given.
- **Memory is bounded by the largest token**: a string or a number is held whole while it is read,
  and nothing else of the text is kept once its event is given.

## Writing

- `render` is the compact text, with no white space: `{"a":[1,2]}`.
- `pretty(indent = 2)` puts each element and member on its own line, `indent` spaces deeper than its
  container, with a space after each member's colon; an empty array or object stays `[]` or `{}`.
  `pretty(0)` is `render`.
- `renderAscii` is the compact text with every character beyond ASCII written as a `\u` escape,
  one beyond U+FFFF as its surrogate pair.
- `writeTo(out: Writer)` writes the compact text to a writer, and `quote(s, ascii = false)` is one
  string as a JSON string literal.

A string is written with `"`, `\` and the control characters below U+0020 escaped, as `\b`, `\f`,
`\n`, `\r` and `\t` where JSON names them and as `\u00XX` otherwise; `/` and every other character
are written as they are. A `Long` is written as its digits. A `Real` is written as the fewest
digits that read back as the same value, and always with a fraction or an exponent, so it reads back
as a `Real`: `2.5`, `1.0`, `-0.0`, `100000000000000000000.0`. From 1e-6 up to 1e21 it is written out
in full, and otherwise with an exponent: `1e+21`, `1.5e-7`, `5e-324`. A NaN or an infinity, which
JSON cannot write, is written `null`. So `parse(j.render)` is `j` for every value with no NaN or
infinity in it, as are `pretty` and `renderAscii` read back, and the text read is the text rendered
for any document already written compactly.

## Taking a value apart

| To | Write |
| --- | --- |
| A member, or None | `j.get("key")`, the last member where the key is written twice |
| An element, or None | `j.get(i)`, None outside the array |
| A member or element that must be there | `j("key")`, `j(i)`, which trap naming the key or index when it is not |
| A field's value of a kind | `asBool`, `asString`, `asVector`, `asMembers`, each an `Option` |
| A number | `asLong`: a `Long`, or a `Real` that is a whole number within a `long` (`3.0`, `1e3`); `asInt` where that fits an `int`; `asReal`: a `Real`, or a `Long` rounded to the nearest `real` |
| Whether it is null, or has a member | `isNull`, `has("key")` |
| An object's keys, the size of an array or object | `keys`, `size` (0 for any other value) |
| An object with a member set | `j.with("key", value)`: the member replaced in place, or added at the end; any other value is returned unchanged |

The accessors are Options so that they chain with `?` in a function answering an `Option`:

```avin
exitCode(event: Json) -> Option[int] = event.get("result")?.get("exit")?.asInt
```

## A program's own types

A type that says `derives JsonEncoder, JsonDecoder` is written and read as JSON with no tree in
between: its encoder writes through a `JsonWriter` and its decoder reads an event at a time through
the [streaming `JsonReader`](#reading-a-stream) above, each written for the type's own fields by the
templates of `dev/avinlang/json/codec.avin`, written as the Avin specification's derivation chapter
describes. There is one reader: a decoder reads text in hand, text arriving from a file or a
connection, and a `Json` value already in hand the same way (see [From and to a `Json`
value](#from-and-to-a-json-value)).

```avin
import dev.avinlang.json
import dev.avinlang.json.{JsonEncoder, JsonDecoder, JsonKey, JsonTag}

data Address(street: string, city: string) derives JsonEncoder, JsonDecoder

data Person(@JsonKey("full_name") name: string, age: int, home: Option[Address]) derives JsonEncoder, JsonDecoder

@JsonTag("kind")
data Shape derives JsonEncoder, JsonDecoder
  Circle(radius: real)
  Empty

main()
  println(json.encode(Person("Ada", 36, None)))     // {"full_name":"Ada","age":36,"home":null}
  println(json.encode[Shape](Circle(1.5)))         // {"kind":"Circle","radius":1.5}
  println(json.decode[Person]("{\"full_name\": \"Ada\", \"age\": 36, \"home\": {\"street\": 1}}").err.get)
  // home.street: expected a string, found a number at line 1, column 52
```

| A value of | Is written as |
| --- | --- |
| A data type or a class deriving them | An object of its fields by name (a class's constructor parameters), in order; a type of several variants writes its variant's name first under `"type"`, or its `@JsonTag`'s key |
| `bool`, `int`, `long`, `real`, `string` | `true`/`false`, a number, a string; a NaN or an infinity as `null` |
| `Option[A]` | `null` for `None`, and the value otherwise |
| `Vector[A]`, `List[A]` | An array |
| `HashMap[K, A]`, `TreeMap[K, A]` | An object, in the map's order; `K` is a `JsonMapKey`: a `string`, or an `int` or a `long` written as its digits |
| `Json` | Itself |

- **`json.encode(value)`** is the compact text; **`json.decode[A](text, maxDepth)`** the value, with
  nothing but white space around it, or a `JsonError`. **`json.decode[A](source: Reader, maxDepth)`**
  reads it as it arrives, a piece at a time, from a file, a connection or any other `Reader`, with the
  same errors at the same places however the source cuts the text; a failure of the source to read is
  the error `the text could not be read: …`, placed where the reading stopped.
- **Decoding** reads fields in any order and skips a key no field has, whatever its value. A field
  absent from the input takes its default, or `None` for an `Option`; any other is the error
  *missing field*. A field written twice is an error, as is a variant name the type does not have
  and a discriminator that is not the object's first key.
- **A number** is read as an `int` or a `long` where it is a whole number in range, written plainly
  or as `3.0` or `1e3`; any other is an error naming the number. A `real` takes any number, and no
  `null`, so a NaN written out does not read back.
- **An error's `path`** is the steps from the document's top to the value at fault, written
  `items[2].name` by `pathText()` and before the message by its `Display`; an error of `parse` has
  none. Its `line`, `column` and `offset` are `long`s, as the stream counts them. Arrays and objects
  nest no deeper than `maxDepth`, 512 unless given, counting those of a value the decoder skips.
- **A fault in the text** met while decoding is the stream's fault, worded as `parse` words it
  (`unexpected '2' in an array, where ',' or ']' was expected`), with the path of the value being
  read where there is one.
- `@JsonKey(name)` renames a field or a variant, `@JsonTag(key)` names a type's discriminator key,
  and `@JsonOmit` leaves a field unwritten and unread, so a decoded value takes its default.
- **`@JsonOmitNone` leaves a field out where its value is `None`**, rather than writing `null`, and
  writes it otherwise; **`json.encode(value, omitNone = true)`** does so for every field of every
  object written, as if each were marked. A `None` that is not a field, an array's element or the
  whole value, is still `null`. Since a field absent from the input is `None`, or its default where
  it has one, what is written reads back. An `Option[Option[A]]` field's `Some(None)` is written
  `null`, and reads back as `None`.
- **`json.decode[A](bytes: Array[byte], maxDepth)`** reads UTF-8 text given as bytes, bytes that are
  not UTF-8 an error at the first of them.
- A type deriving them where a field's type lacks them is an error at the `derives` entry naming the
  field. A union has an encoder where its members do, and no decoder; an open class has an encoder
  and no decoder, since a value of its type could be of any subclass; a field typed as the `Map`
  trait has neither, its `HashMap` or `TreeMap` type both.

## From and to a `Json` value

```text
decode[A: JsonDecoder](tree: Json, maxDepth: int = defaultMaxDepth) -> Result[A, JsonError]
toTree[A: JsonEncoder](value: A, omitNone: bool = false) -> Json
JsonReader.fromTree(tree: Json, maxDepth: int = defaultMaxDepth) -> JsonReader
```

A program that has parsed a message to look at its `method` decodes the part it wants from the tree,
without reading the text again: `json.decode[Params](message("params"))`. The decoder is the one that
reads text, so every shape that decodes from text decodes from a tree, to the same value or the same
error. An error's path starts at the value given, so decoding a message's `params` names
`textDocument.uri`, where decoding the whole message names `params.textDocument.uri`. A tree has no
line or column: such an error's `line`, `column` and `offset` are 0, and it displays as its path and
message, `textDocument.uri: expected a string, found a number`.

`toTree(value)` is the `Json` that `encode(value)` writes, member for member in the same order, so
`s"${json.toTree(v)}"` is `json.encode(v)` and `json.parse(json.encode(v))` is `Ok(json.toTree(v))`.
A NaN or an infinity, which text writes as `null`, is `Null`. `omitNone` is as in `encode`.
`JsonWriter.toTree()` is the writer behind it, whose `tree()` is the value written.

## Over HTTP

The module **`dev.avinlang.json.http`** joins the derived codecs to the standard library's HTTP
server, `avin.net.http`:

```avin
import avin.net.http.{Response, Route, Router, ServerRequest}
import dev.avinlang.json.{JsonDecoder, JsonEncoder}
import dev.avinlang.json.http.jsonResponse

data NewTodo(title: string, notes: Option[string] = None) derives JsonEncoder, JsonDecoder

create(request: ServerRequest) -> Response = request.jsonBody[NewTodo]() match
  Ok(todo) -> jsonResponse(201, todo, omitNone = true)
  Err(answer) -> answer

val router = Router([Route.post("/todos", create)])
```

| Call | Gives |
| --- | --- |
| `jsonResponse(status, value, omitNone = false)` | A response of `status` whose body is `value` encoded as compact JSON, of the type `application/json` |
| `request.decodeJson[A](maxDepth = defaultMaxDepth)` | The body decoded as an `A`, or a `BodyError` |
| `request.jsonBody[A](maxDepth = defaultMaxDepth)` | The body decoded as an `A`, or the `Response` a client is answered, so a handler answering a `Result` takes it with `?` |
| `isJsonType(contentType)` | Whether a `Content-Type` is JSON's |

- **The request's `Content-Type` must be JSON's**: `application/json` or a type `application/…+json`,
  with any parameters and without regard to case. Any other type, or none, is the `BodyError`
  `UnsupportedType(found)`, answered `415`.
- **A body that does not decode** is `Malformed(error)`, the decoder's `JsonError`, answered `400`.
  The body is read as UTF-8 bytes, so bytes that are not UTF-8 are an error rather than read as
  replacement characters.
- **The answer is a JSON object**: `error`, a `detail` saying what is wrong and, for a body that does
  not decode, the `path` to the value at fault where there is one, and its `line` and `column`:

```text
{"error":"invalid JSON body","detail":"expected a string, found a number","path":"home.street","line":2,"column":21}
{"error":"unsupported media type","detail":"expected application/json, found text/plain"}
```

A `BodyError`'s `status` and `response` give its status and that answer.

## Tests

`avin test` in this directory runs the package's tests, in `tests/`; `tests/readme.avin` holds this
page's examples; `tests/tree.avin` holds decoding from and encoding to a `Json` value, each shape of
type checked against its text.

Decision: decoding from a tree is a `JsonReader` over the tree, not a second decoder, so a derived
decoder has one path whatever the source; and the encoder to a tree is the one encoder writing to a
`JsonWriter` that builds a value, named `toTree` because `encode` is already the text.
