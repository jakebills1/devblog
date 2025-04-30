+++
date = '2025-04-30T10:11:51-06:00'
draft = true
title = 'Redis in Ruby Part 2: Persistence'
+++

This is a followup to my [last post](/posts/redis-in-ruby) about implementing the basic functionality of a Redis server in Ruby. The short version of that is that Redis is an in-memory key-value store and interacting with it looks something like this:
```bash-session
$ redis-cli SET foo bar
  OK
$ redis-cli GET foo
  "bar"
```

## Persistence
Since Redis is an in memory DB, when it shuts down all the data that has been saved is lost by default. But Redis also implements a [SAVE command ](https://redis.io/docs/latest/commands/save/) that saves all of the key-value pairs currently in the database to a file with a `.rdb` extension.  Redis servers can be passed the path to an `.rdb` file as part of their configuration and will load it's data on startup. 

`.rdb` files also play a part in replication. When a Redis server that is configured to be a replica of another server is started up, it will receive that server's data as an `.rdb` file to sync them up. 

### Parsing an rdb file
`rdb` files are not normal files that you can open and see what different values are saved to the database, they are written in binary according to a special format. Using `hexdump` to view the contents of a simple rdb file looks something like this:
```bash-session
$ redis-cli flushdb
  OK
$ redis-cli SET foo bar
  OK
$ redis-cli SAVE
  OK
$ redis-cli CONFIG GET dir
  1) "dir"
  2) "/opt/homebrew/var/db/redis"
$ hexdump -C /opt/homebrew/var/db/redis/dump.rdb
```
```text
00  52 45 44 49 53 30 30 31  31 fa 09 72 65 64 69 73 |REDIS0011..redis|
10  2d 76 65 72 05 37 2e 32  2e 36 fa 0a 72 65 64 69 |-ver.7.2.6..redi|
20  73 2d 62 69 74 73 c0 40  fa 05 63 74 69 6d 65 c2 |s-bits.@..ctime.|
30  98 c8 0f 68 fa 08 75 73  65 64 2d 6d 65 6d c2 20 |...h..used-mem. |
40  6e 12 00 fa 08 61 6f 66  2d 62 61 73 65 c0 00 fe |n....aof-base...|
50  00 fb 01 00 00 03 66 6f  6f 03 62 61 72 ff 26 8c |......foo.bar.&.|
60  44 c2 e9 88 0e 97                                |D.....|
66
```

For those new to the `hexdump` tool (the `-C` switch sets the display to both hex and ASCII), the columns are as follows:
- The byte offset, indicating the byte index each line begins at from 0 to the number of bytes of the file - 1. This is in hexadecimal, so each line is 16 bytes long
- The hexadecimal representation of the actual data in the file. Each 2 digit hex number is one byte, and they correspond to the first column meaning each line is up to 16 bytes long
- An interpretation of the data. Some of the data can be interpreted as ascii, and that is shown in the last column. The key value pair `foo, bar` is located at the offset 86

A substantial explanation of the different parts of an `rdb` file [is available here,](https://rdb.fnordig.de/file_format.html) but in a nutshell the file consists of:
- A 9 byte header consisting of the ASCII codes for the string `"REDIS00nn"` where `nn` is the rdb version number the file was produced under
- Metadata including the Redis server version that produced the file, timestamp, memory info, etc
- A list of the different databases included in the dump. For each database:
	- The sizes of the hash tables representing the key-value pairs that were persisted in the database
	- The key-value pairs expiring at a certain time in seconds, if present
	- The key-value pairs expiring at a certain time in milliseconds, if present
	- The key-value pairs with no expiration, if any
- A checksum

Each section, apart from the header, is preceded with a one-byte value called an opcode which indicates the type of section upcoming. For example the hex value `FE` indicates the start of the database list and is found at offset 79 in the `hexdump` output above.

#### Datatypes
For the sake of storing more data in fewer bytes, not all of the data in an `rdb` file is encodable as ascii. The format uses some custom encodings, which will explain briefly:
##### Length Prefixed Strings
Length encoding is the technique used to store the length of an object in the stream preceding it's value. For example, to read a length prefixed string, you read the length first, then use the result to read the bytes that make up the string. A simplified example of reading a length prefixed string:
```ruby
bytes = StringIO.new "\x03\x66\x6f\x6f"
# => #<StringIO:0x00000001206fe188>
length = bytes.read(1).unpack1('C')
# => 3
val = bytes.read(length)
# => "foo"
```
##### Integers
Integers are similar to length prefixed strings in that their length is encoded by the byte preceding them. However, reading that byte is a bit more complicated. Here is an example of reading the bitstream starting at offset 63 that encodes a 32 bit integer:
```ruby
# the 2 most significant bits in the 
# first byte indicates how to read the length
bytes = StringIO.new "\xc2\x98\xc8\x0f\x68"
# => #<StringIO:0x00000001208718f8>
length_enc = bytes.read(1).unpack1('C') 
# => 194

# shift the value over 6 bits to 
# isolate the 2 most significant bits
msb = length_enc >> 6 
# => 3
width, format = if msb == 0b11
                  # extract the 6 least significant bits
                  case length_enc & 0b00111111 
                  when 0 # 0000 in binary
                    [1, 'C']
                  when 1 # 0001 in binary
                    [2, 'S']
                  when 2 # 0010 in binary
                    [4, 'L']
                  end
                else 
                  # skipping other cases for brevity
                end
val = bytes.read(width).unpack1(format)
# => 1745864856
```

You may notice some unfamiliar binary tricks in the code above:
- Bit shifting: bitstreams can be shifted to the left or right, and in this code I shifted the bitstream right 6 bits to isolate the most significant bits of the first byte read
- Binary AND ( `&` ): ANDing together 2 bit streams results in a 3rd bit stream where a bit at index i is 1 if both input bit streams at i were 1. I use it in this example to get the 6 least significant bits out of the byte read on line 2

## Conclusion
To see how this all comes together, see my solution [here.](https://github.com/jakebills1/codecrafters-redis-ruby/blob/b8cd43ff130cca9e891f242a5cb6699950c367e8/lib/redis/rdb_parser.rb) Redis has many more data types like lists and maps, each with its own encoding. My implementation doesn't support these yet, but I have learned a lot about binary protocols and encoding.

## References
- [RDB Format Reference](https://rdb.fnordig.de/file_format.html)