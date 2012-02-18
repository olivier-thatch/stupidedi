# Stupidedi

![Screenshot](https://raw.github.com/kputnam/stupidedi/master/doc/images/edi-pp.png)

* [GitHub project](http://github.com/kputnam/stupidedi)
* [Human Documentation](https://github.com/kputnam/stupidedi/tree/master/doc)
* [API Documentation](http://rubydoc.info/github/kputnam/stupidedi/master/frames)

Stupidedi is a high-quality library for parsing, generating, validating,
and manipulating ASC X12 EDI documents. Very roughly, it's jQuery for
EDI.

For those unfamiliar with ASC X12 EDI, it is a data format used to
encode common business documents like purchase orders, delivery
notices, and health care claims. It is similar to XML in some ways,
but precedes it by about 15 years; so if you think XML sucks, you
will love to hate EDI.

## What problem does it solve?

Transaction set specifications can be enormous, boring, and vague.
Trading partners can demand strict adherence (often to their own unique
interpretation of the specification) of the documents you generate.
However, documents they generate themselves are often non-standard
and require flexibility to parse them.

Stupidedi enables you to encode these transaction set specifications
directly in Ruby. From these specifications, it will generate a parser
to read incoming messages and a DSL to generate outgoing messages. This
approach has a huge advantage over writing a parser from scratch, which
can be error-prone and difficult to change.

Significant thought was put into the design of the library. Some of
the features are described here.

### Robust tokenization and parsing

Delimiters, line breaks, and out-of-band data between interchanges are
handled correctly. While many trading partners follow common conventions,
it only takes one unexpected deviation, like swapping the ":" and "~"
delimiters, to render a hand-written parser broken.

Stupidedi handles many edge cases that can only be anticipated by reading
carefully between the lines of the X12 documentation.

### Instant feedback on error conditions

When generating EDI documents, validation is performed incrementally
on each segment. This means the instant your client code violates the
specification, an exception is thrown with a meaningful stack trace.
Other libraries only perform validation after the entire document has
been generated, while some don't perform validation at all.

Stupidedi performs extensive validation and ensures well-formedness.
See the human readable documentation in doc/Generating.md for more
details.

### Encourages readable client code

Unlike other libraries, generating documents doesn't involve memorizing
noisy identifiers from the specification (like C001 or LOOP2000), which
identify elements of the grammar that don't actually appear in the output.

Like HAML or BuilderXML, the DSL for generating documents closely matches
terminology from the problem domain. You can see in the example below that
code looks very similar to an EDI document (without the noisy delimiters).
This makes it easy to understand, assuming a reasonable familiarity with
EDI.

### Efficient parsing and traversing

The API is designed almost exclusively using immutable data structures,
making it thread-safe for runtimes that can utilize multiple cores. While
immutability places higher demand on garbage collection, Stupidedi has
been profiled and tuned to minimize the more significant bottlenecks.

### Helps developers gain familiarity

## Why not a commercial translator?

Commercial EDI translators solve a different set of problems. Many focus
on translating between EDI and another data format, like XML, CSV, or a
relational database. This isn't particularly productive, as you still have
to unserialize the data to do anything with it.

## What doesn't it solve?

It isn't a translator. It doesn't have bells and whistles, like the
commercial EDI translators have, so it...

* Doesn't convert to/from XML, CSV, etc
* Doesn't transmit or receive files
* Doesn't do encryption
* Doesn't connect to your database
* Doesn't queue messages for delivery or receipt
* Doesn't generate acknowledgements
* Doesn't have a graphical interface

These can be accomplished using other libraries or your own code.

## Alternative libraries

Stupidedi is an opinionated library, and maybe you don't agree with
it. Here are a few alternative libraries:

* http://www.appdesign.com/x12parser/
* http://edi4r.rubyforge.org/edi4r/
* http://edival.sourceforge.net/
* http://www.edidev.com/

## Examples

### Utilities

Pretty print the syntax tree

    $ ./bin/edi-pp spec/fixtures/X222-HC837/1-good.txt
    ...
            TableVal[Table 3 - Summary](
              SegmentVal[SE: Transaction Set Trailer](
                Nn.value[  E96: Number of Included Segments](45),
                AN.value[ E329: Transaction Set Control Number](0021)))), 
          SegmentVal[GE: Functional Group Trailer](
            Nn.value[  E97: Number of Transaction Sets Included](1),
            Nn.value[  E28: Group Control Number](1))), 
        SegmentVal[IEA: Interchange Control Trailer](
          Nn.value[  I16: Number of Included Functional Groups](1),
          Nn.value[  I12: Interchange Control Number](905))))
    49 segments
    49 segments
    0.140 seconds

Perform validation on a file

    $ ./bin/edi-ed spec/fixtures/X222-HC837/1-bad.txt
    [AK905(file spec/fixtures/X222-HC837/1-bad.txt,
           line 16, column 4, is not an allowed value,
           ID.value[ E479: Functional Identifier Code](XX)),
     IK304(file spec/fixtures/X222-HC837/1-bad.txt,
           line 33, column 1,
           missing N4 segment, NM1),
     IK304(file spec/fixtures/X222-HC837/1-bad.txt,
           line 35, column 1,
           missing N4 segment, NM1)]
    46 segments
    0.177 seconds

### Generating, Writing

```ruby
require "stupidedi"

# You can customize this to delegate to your own grammar definitions, if needed.
config = Stupidedi::Config.default

b = Stupidedi::Builder::BuilderDsl.build(config)

# These methods perform error checking: number of elements, element types, min/max
# length requirements, conditionally required elements, valid segments, number of
# segment occurrences, number of loop occurrences, etc.
b.ISA("00", "", "00", "",
      "ZZ", "SUBMITTER ID",
      "ZZ", "RECEIVER ID",
      "990531", "1230", nil, "00501", "123456789", "1", "T", nil)

# The API tracks the current position in the specification (e.g., the current loop,
# table, etc) to ensure well-formedness as each segment is generated.
b. GS("HC", "SENDER ID", "RECEIVER ID", "19990531", "1230", "1", "X", "005010X222")

# The `b.default` value can be used to generate the appropriate value if it can
# be unambigously inferred from the grammar.
b. ST("837", "1234", b.default)

# You can use string representations of data or standard Ruby data types, like Time.
b.BHT("0019", "00", "X"*30, "19990531", Time.now.utc, "CH")
b.NM1(b.default, "1", "PREMIER BILLING SERVICE", "", "", "", "", "46", "12EEER000TY")
b.PER("IC", "JERRY THE CLOWN", "TE", "3056660000")

b.NM1("40", "2", "REPRICER JONES", "", "", "", "", "46", "66783JJT")
b. HL("1", "", "20", "1")

b.NM1("85", "2", "PREMIER BILLING SERVICE", "", "", "", "", "XX", "123234560")
b. N3("1234 SEAWAY ST")
b. N4("MIAMI", "FL", "331111234")
b.REF("EI", "123667894")
b.PER("IC", b.blank, "TE", "3056661111")

b.NM1("87", "2")
b. N3("2345 OCEAN BLVD")
b. N4("MIAMI", "FL", "33111")
b. HL("2", "1", "22", "0")
b.SBR("S", "18", "", "", "12", "", "", "", "MB")

b.NM1("IL", "1", "BACON", "KEVIN", "", "", "", "MI", "222334444")
b. N3("236 N MAIN ST")
b. N4("MIAMI", "FL", "33413")
b.DMG("D8", "19431022", "F")

b.machine.zipper.tap do |z|
  # The :component, and :repitition parameters can also be specified as elements
  # of the ISA segment, at `b.ISA(...)` above. When generating a document from
  # scratch, :segment and :element must be specified -- if you've parsed the doc
  # from a file, these params will default to whatever was used in the file, or
  # you can override them here.
  separators =
    Stupidedi::Reader::Separators.build(:segment    => "~\n",
                                        :element    => "*",
                                        :component  => ":",
                                        :repetition => "^")

  # You can also serialize any subtree within the document (e.g., everything inside
  # some ST..SE transaction set, or a single loop.
  w = Stupidedi::Writer::Default.new(z.root, separators)
  w.write($stdout)
end
```

### Reading, Traversing

```ruby
require "stupidedi"

config = Stupidedi::Config.default
parser = Stupidedi::Builder::StateMachine.build(config)

input  = if RUBY_VERSION > "1.8"
           File.open("spec/fixtures/X221-HP835/1-good.txt", :encoding => "ISO-8859-1")
         else
           File.open("spec/fixtures/X221-HP835/1-good.txt")
         end

# Reader.build accepts IO (File), String, and DelegateInput
parser, result = parser.read(Stupidedi::Reader.build(input))

# Report fatal tokenizer failures
if result.fatal?
  result.explain{|reason| raise reason + " at #{result.position.inspect}" }
end

def el(m, *ns, &block)
  if Stupidedi::Either === m
    m.tap{|m| el(m, *ns, &block) }
  else
    yield(*ns.map{|n| m.element(*n).map(&:node).map(&:value).fetch(nil) })
  end
end

# Print some information
parser.first
  .flatmap{|m| m.find(:GS) }
  .flatmap{|m| m.find(:ST) }
  .tap do |m|
    el(m.find(:N1, "PR"), 2){|e| puts "Payer: #{e}" }
    el(m.find(:N1, "PE"), 2){|e| puts "Payee: #{e}" }
  end
  .flatmap{|m| m.find(:LX) }
  .flatmap{|m| m.find(:CLP) }
  .flatmap{|m| m.find(:NM1, "QC") }
  .tap{|m| el(m, 3, 4){|l,f| puts "Patient: #{l}, #{f}" }}
```