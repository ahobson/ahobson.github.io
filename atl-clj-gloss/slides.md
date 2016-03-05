# gloss-talk

!SLIDE

# atl-clj `gloss`

## ahobson@gmail.com

!SLIDE left light

## What is gloss?

* <https://github.com/ztellman/gloss>
* A library for encoding and decoding raw bytes.
  
}}} images/glossy.jpg::sevenbrane::flickr::https://flickr.com/sevenbrane

!SLIDE left light

## What is gloss?

* Useful for parsing network protocols and custom file formats.
* I won't cover the full details in this presentation. The wiki page
  is pretty good, but some functions are not covered there. UTSL.
  
}}} images/glossy.jpg::sevenbrane::flickr::https://flickr.com/sevenbrane

!SLIDE left

## DNS Concepts

* I'm going to cover gloss using the example of parsing and generating
  DNS traffic.
* <https://tools.ietf.org/html/rfc1035>
* NOTE: Does not handle label compression, but I think it could.

!SLIDE left

## DNS Concepts

* NAME/QNAME: The name in DNS.
* LABEL: Parts of a NAME (e.g. "foo", "example").
* TYPE/QTYPE: Record Type / Query Type (e.g "A record" for IPv4).
* TTL: Time To Live.
* RR: Resource Record or "DNS Entry".
* RDATA: Resource DATA describing the RR, different depending on TYPE.

``` text
;; NAME     TTL CLASS TYPE  RDATA
foo.example 300 IN    A     10.2.3.4
bar.example 300 IN    CNAME foo.example
```

!SLIDE left glow

## Gloss Concepts

* A byte format description is called a `frame`.

}}} images/frame.jpg::jewellshoots::flickr::https://flickr.com/jewellshoots

!SLIDE left

## Gloss Concepts

* A byte format description is called a `frame`.
* Compiling a `frame` with `compile-file` returns a `codec` that can
  be used for encoding and decoding bytes.
* `defcodec` is a macro that calls `compile-frame`.

!SLIDE left glow

## Gloss Primitives

}}} images/primative.jpg::oneras::flickr::https://flickr.com/oneras

!SLIDE left

## Gloss Primitives

* Primitive types: `:byte`, `:int16`, `:int32`, `:int64`, `:float32`,
  `:float64`, `:ubyte`, `:uint16`, `:uint32`, and `:uint64`.
* Big/Little Endian: e.g. `:int32-be` or `:int32-le`.

!SLIDE left bigcode

## Gloss encoding

* Gloss encodes to a sequence of `ByteBuffer`s.

``` clojure
dns-clj.protocol> (gio/encode :uint32-be 0x01020304)
(#object[java.nio.HeapByteBuffer 0x227330c
 "java.nio.HeapByteBuffer[pos=0 lim=4 cap=4]"])
dns-clj.protocol> (defn gloss-inspect [a]
                        (map #(vec (.array %)) a))
dns-clj.protocol> (gloss-inspect
                    (gio/encode :uint32-be 0x01020304))
([1 2 3 4])
```

!SLIDE left glow

## Gloss Strings

* Requires specifying a character set.
* Can have a static length, delimiters (or both).

}}} images/strings.jpg::ddrmaxgt37::flickr::https://flickr.com/ddrmaxgt37

!SLIDE left bigcode

## Gloss Strings

```clojure
(defcodec command (glc/string :utf-8))
;; or maybe
(defcodec short-command (glc/string :utf-8 :length 3))
;; or maybe
(defcodec line-command (glc/string :utf-8 :delimiters
                                     ["\r" "\r\n"]))
```

```clojure
dns-clj.protocol> (gloss-inspect (gio/encode
                                   command "foo"))
([102 111 111])
dns-clj.protocol> (gloss-inspect (gio/encode
                                   short-command "foo"))
([102 111 111])
dns-clj.protocol> (gloss-inspect (gio/encode
                                   line-command "foo"))
([102 111 111] [13])
```

!SLIDE left bigcode

## Gloss decoding

* Gloss decodes anything that can be converted into a byte buffer.
* Gloss wants to enforce that a codec consumes all input.

```clojure
dns-clj.protocol> (gio/decode short-command
                    (byte-array [102 111 111 111]))
Exception Bytes left over after decoding frame.
  gloss.io/decode (io.clj:87)
dns-clj.protocol> (gio/decode short-command
                    (byte-array [102 111 111 111]) false)
"foo"
```

!SLIDE left bigcode

## Gloss `enum`

* Map values to symbols.

```clojure
(defcodec qtypes
  (glc/enum :uint16-be {:a 1
                        :cname 5
                        :any 255}))
```

``` clojure
dns-clj.protocol> (gio/encode qtypes :a)
(#object[java.nio.HeapByteBuffer 0x5baba097
"java.nio.HeapByteBuffer[pos=0 lim=2 cap=2]"])
dns-clj.protocol> (gloss-inspect *1)
([0 1])
dns-clj.protocol> (gio/decode qtypes *2)
:a
```
!SLIDE left

## DNS QNAME

* From <https://tools.ietf.org/html/rfc1035#section-4.1.2>


> a domain name represented as a sequence of labels, where each
> label consists of a length octet followed by that number of
> octets. The domain name terminates with the zero length octet for
> the null label of the root. Note that this field may be an odd
> number of octets; no padding is used.

!SLIDE left glow

## Gloss `finite-frame`

* A frame prefixed by its size.

}}}images/finite.jpg::gadl::flickr::https://flickr.com/gadl

!SLIDE left bigcode

## Gloss `finite-frame`

* A frame prefixed by its size.

```clojure
(defcodec label
  (glc/finite-frame :byte (glc/string :ascii)))
```

```clojure
dns-clj.protocol> (gloss-inspect (gio/encode label "foo"))
([3] [102 111 111])
```

!SLIDE left glow

## Gloss `repeated`

* A dynamically sized sequence with either a prefix size or ending
  delimiter(s).
  
}}}images/repeated.jpg::bottspot::flickr::https://flickr.com/bottspot

!SLIDE left bigcode

## Gloss `repeated`

* A dynamically sized sequence with either a prefix size or ending
  delimiter(s).
  
```clojure
(glc/repeated label :delimiters [(byte 0)])
```

```clojure
dns-clj.protocol> (gloss-inspect (gio/encode
  (glc/repeated label :delimiters [(byte 0)])
    ["foo" "example"]))
([3] [102 111 111] [7] [101 120 97 109 112 108 101] [0])
```

!SLIDE left glow

## Gloss `compile-frame`

* Takes an optional `pre-encoder` and `post-encoder` to transform the
  data before/after encoding/decoding.
  
}}}images/xform.jpg::tomswift::flickr::https://flickr.com/tomswift

!SLIDE left bigcode

## Gloss `compile-frame`

* Takes an optional `pre-encoder` and `post-encoder` to transform the
  data before/after encoding/decoding.

``` clojure
;; assume these are defined
(defn string->label-array [s] (str/split s #"\."))
(defn label-array->string [a] (str/join "." a))

(defcodec qname
  (glc/compile-frame
   (glc/repeated label :delimiters [(byte 0)])
   string->label-array
   label-array->string))
```

```clojure
dns-clj.protocol> (gloss-inspect
                    (gio/encode qname "foo.com"))
([3] [102 111 111] [3] [99 111 109] [0])
```

!SLIDE left

## DNS RR Format

* From <https://tools.ietf.org/html/rfc1035#section-3.2.1>
```text
All RRs have the same top level format shown below:

                                    1  1  1  1  1  1
      0  1  2  3  4  5  6  7  8  9  0  1  2  3  4  5
    +--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+
    |                                               |
    /                                               /
    /                      NAME                     /
    |                                               |
    +--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+
    |                      TYPE                     |
    +--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+
    |                     CLASS                     |
    +--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+
    |                      TTL                      |
    |                                               |
    +--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+
    |                   RDLENGTH                    |
    +--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--|
    /                     RDATA                     /
    /                                               /
    +--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+
```

!SLIDE left glow

## Gloss `ordered-map`

* Frames are by default encoded in a stable, but unspecified order.

}}}images/order.jpg::s_y_s::flickr::https://flickr.com/s_y_s

!SLIDE left bigcode

## Gloss `ordered-map`

* Frames are by default encoded in a stable, but unspecified order.
* Works for gloss <-> gloss interop, but sometimes we need to specify
  the order.

``` clojure
(defcodec answer-class-ttl
  (glc/ordered-map :name qname
                   :type qtypes
                   :class class-types
                   :ttl :uint32-be))
```

!SLIDE left

## Gloss `header`

* A frame that defines the following frame.
* Takes a frame, a function to return the following frame given the
parsed header, and then for encoding, a function that takes the data
and returns a header.


!SLIDE left bigcode

## Gloss `header`

```clojure
(defcodec rdata-a
  (glc/finite-frame :uint16-be
    (glc/ordered-map :address :uint32-be)))
(defcodec rdata-c
  (glc/finite-frame :uint16-be
    (glc/ordered-map :cname qname)))
(def answer-codecs
  {:a     rdata-a
   :cname rdata-c})
;; Ignore `build-merge-header-with-data` for now
(defcodec rrec
  (glc/header
    answer-class-ttl
    (build-merge-header-with-data
      (fn [h] (get answer-codecs (:type h))))
    (fn [b]
      (select-keys b [:name :type :class :ttl]))))
```

!SLIDE left

### DNS header section format
* <https://tools.ietf.org/html/rfc1035#section-4.1.1>

```text
The header contains the following fields:

                                    1  1  1  1  1  1
      0  1  2  3  4  5  6  7  8  9  0  1  2  3  4  5
    +--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+
    |                      ID                       |
    +--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+
    |QR|   Opcode  |AA|TC|RD|RA|   Z    |   RCODE   |
    +--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+
    |                    QDCOUNT                    |
    +--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+
    |                    ANCOUNT                    |
    +--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+
    |                    NSCOUNT                    |
    +--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+
    |                    ARCOUNT                    |
    +--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+
```

!SLIDE left bigcode
### Gloss `bit-map`

```clojure
(defcodec header
  (glc/ordered-map :id :uint16-le
                   :options (glc/bit-map :qr 1
                                         :opcode 4
                                         :aa 1
                                         :tc 1
                                         :rd 1
                                         :ra 1
                                         :z 3
                                         :rcode 4)
                   :qdcount :uint16-be
                   :ancount :uint16-be
                   :nscount :uint16-be
                   :arcount :uint16-be))
```

!SLIDE left bigcode

### Simple server using `aleph`

* <https://github.com/ztellman/aleph>
* Easy to hook up `gloss` and `aleph`.

```clojure
(def answer-db
  {"foo.example" [{:type :a     :class :in :ttl 300
                   :name "foo.example"
                   :address 0x01020304}]
   "bar.example" [{:type :cname :class :in :ttl 300
                   :name "bar.example"
                   :cname "foo.example"}]})
;; ...

(defn answerer
  "Answer `msg` on manifold stream `s`."
  [s msg]
  (let [d (gio/decode dnsp/dns-msg (:message msg))
        resp (build-response d)]
    (ms/put! @s (assoc msg :message
                  (gio/encode dnsp/dns-msg resp)))))
```

!SLIDE left bigcode

### Demo!

```clojure
dns-clj.core> (def svr (start-server 5399))
#'dns-clj.core/svr
;; Do some queries
;; dig -p 5399 @127.0.0.1 foo.example
;; dig -p 5399 @127.0.0.1 cname bar.example
;;
dns-clj.core> (ms/close! @svr)
true
```

!SLIDE

### Questions?

* <https://github.com/ahobson/dns-clj>
* <https://ahobson.github.io/atl-clj-gloss>

