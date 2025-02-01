# Motivation
There are enumerable options for serialization and configuration file formats in the current landscape. However, none of them seem to scratch my itch due to them being what I consider a swing and a miss or having their own issues. This won't be a discussion about all of the different formats. We will however reference relevant points of existing formats to justify the motiviations behind the decisions of a new json inspired format. The new format is coined jsonp (json plus), for lack of a better name.

The reality around json is that despite its intent and roots of being a data only serialization format, it has spread to being used as a configuration format in an enormous amount of projects. As a result, we should try and update the format for that use case as I believe json has its strengths and weaknesses. Json is a great choice for serialization as it is dirt simple to parse. On the other hand, it lacks features for human consumption as a configuration file format due to this. It is possible to extend json for configuration, but like many things you can't have the cake and eat it too. If we extend it for configuration, it porbably won't be dirt simple to parse anymore for serialization purposes. This is the tradeoff. However, I don't feel like the quality of life additions are computationally insurmountable. I would ultimately like a format that is human consumable and serializable to the extent that it is generally performant. 

# Specification
Here we will give a 'soft' specification of the jsonp format. Soft in that its still a work in progress and subject to change. Motiviations behind some decisions will also be given.

# Conventions
1. Unicode codepoints may be specified as UXXXXX where X represents a hexadecimal digit within the specification. For example, U20 is the unicode codepoint for a space character.

# Encoding
The only valid encoding for a jsonp text shall be utf8. No text shall contain a BOM.

Some other json inspired formats seem to want to allow all encodings, such as utf16, utf32, etc.. The reality that I have seen is that 100% of json texts that I have encountered have been utf8 encoded. In fact, during my entire software career, I have yet to encounter any other encoding besides ASCII and utf8 in the wild for texts (json or other) that are stored on disk or streamed. I do occasionally run into utf16 encoding for things like usb, but that is not a stored or streamed text.

# File extension
The file extension for jsonp texts that exist on a storage medium shall be .jsonp

# Backwards compatibility with json
jsonp is a super set of json, like most json derived formats. A jsonp text may not be valid json text; however, a json text shall be a valid jsonp text. This means that a jsonp compliant parser must accept any valid json text.

# Document format
A jsonp document shall contain a single root value similar to a json document. Likewise an empty jsonp document containing no root value is invalid. A stream or series of bytes may contain multiple jsonp documents. This is where the format differs slightly. A jsonp document may be terminated in one of several ways.

## End of text
A jsonp document can be terminated by encountering the end of the stream of bytes that contains the document. This is no different from a json document. Any root value must be properly terminated before the end of the text is encountered. Most primitive values will not require explicit termination. However, objects and arrays must properly be terminated before the end of the text is encountered, otherwise the jsonp document is considered invalid.

## Record terminator (U1E)
A jsonp document can be terminated by encountering a record terminator character within the stream of bytes. This is known as the streaming format and allows multiple jsonp documents (or root level values) within a single stream of bytes. The expected use of this format will be client / server like relationships, but it need not be. In this format, a record terminator **must** be present after each root value regardless of additional values to be sent or the end of text encountered, otherwise the text is deemed invalid. In this manner, the record terminator acts as a terminator rather than a separator and ensures it can be relied upon for ease of streaming.

The jsonl (jsonl) format is similar; however, a line terminator is used instead of a special control character meant for the purpose. This has the major downside that line terminators are no longer allowed within the rest of the text. Thus root values are smooshed onto a single line which can greatly hinder readability.

# Control characters
The control characters that cannot appear anywhere within a jsonp text are the unicode characters in the Cc (control) category. This does not include whitespace, line terminators, or the record terminator. If control characters need to be within the text, they can be embedded within a string using a unicode escape sequence.

# Whitespace
The valid whitespace characters that can appear within a jsonp text are unchanged from json.
1. tab
2. space
3. carriage return
4. line feed

# Line terminators
The valid line terminator sequences that can appear within a text are unchanged from json.
1. CR (carriage return)
2. LF (line feed)
3. CRLF (carriage return followed by line feed)

# Comments
Single line comments can be written using the # symbol. Any codepoint except control characters can appear after the start of the comment. The comment is terminated by encountering a line terminator.

# Structural tokens
A jsonp text contains the same structural tokens as a json document

| Token | Unicode | Name            | Meaning
| ----- | ------- | --------------- | --------------------------------- |
|   {   |   U7B  | Opening brace   | Starts an object value            |
|   }   |   U7D  | Closing brace   | Ends / terminates an object value |
|   [   |   U5B  | Opening bracket | Starts an array value             |
|   ]   |   U5D  | Closing bracket | Ends / terminates an array value  |
|   :   |   U3A  | Colon           | Property key, value separator     |
|   ,   |   U2C  | Comma           | Separates values and properties   |

# Keywords and keyword literals
The following table details the accepted keywords and keyword literals within a jsonp text

| Keyword   | Meaning
| --------- | ----------------------------------------------------
| null      | Represents the null value
| true      | Represents a boolean value of true
| false     | Represents a boolean value of false
| infinity  | Represents the floating point value of infinity
| -infinity | Represents the floating point value of negative infinity
| nan       | Represents the floating point value of not a number

# Number values
Number values are extended beyond json and support additional formats. A number can contain digit seperators after the first digit using the _ character. Refer to keyword and keyword literal section for additional special number values.

| Format      | Specifier | Example                   |
| ----------- | --------- | ------------------------- |
| Binary      | 0b        | 0b11001010, 0b1010_0101   |
| Octal       | 0o        | 0o12345671, 0o123_345     |
| Hexadecimal | 0x        | 0xDEADBEEF, 0xDE_AD_BE_EF |

# Strings
Strings are extended beyond json support and have the following properties
1. Single quoted strings are supported. Double quotes do not need to be escaped within single quoted strings.
2. Double quoted strings are supported. Single quotes do not need to be escaped within double quoted strings.
3. Multiline strings are supported.
4. Additional escape sequences are supported.
5. Unquoted string values are **not** supported.
6. Content within strings can be any codepoint except control characters

## Multiline strings
Multi line strings are supported with no additional syntax. If a line terminator is encountered within a string, it and any subsequent whitespace are consumed until the next codepoint is encountered. Once the next codepoint is encountered, it is concatenated to the string and processing continues.

## Escape sequences
The table lists the escape sequences that are accepted within strings.

| Sequence | Meaning        
| -------- | ---------------
| \b       | backspace
| \f       | form feed
| \t       | tab
| \r       | carriage return 
| \n       | line feed
| '\ '     | Escape sequence for a space character (U5C followed by U20)
| \\\\     | Backslash
| \\'      | Single quote, need not be escaped within a double quoted string
| \\"      | Double quote, need not be escaped within a single quoted string
| \x{2}    | Escape followed by 2 hexadecimal digits. 2 digit short unicode codepoint
| \u{4}    | Escape followed by 4 hexadecimal digits. 4 digit short unicode codepoint or two sequences representing a utf16 surrogate pair
| \U{1,6}  | Escape followed by 1-6 hexadecimal digits. Specifies a full unicode codepoint

### Notes
The space escape sequence can be used on a subsequent line of a multiline string to concatenate a space at the beginning of the next line. The parser will encounter the backslash which stops it from consuming whitespace and starts string concatenation of the next line.

# Separators
Separators are extended beyond json support. The following rules apply.
1. A value can have a single trailing separator regardless of position within a document. The separator is optional and is only used to delimit or terminate the value.
2. A property (key, value) pair can have a single trailing separator regardless of position within a document. The separator is optional and is only used to delimit or terminate the property.

A jsonp compliant parser has to at a minimum use whitespace to delimit or terminate values and properties. A separator encountered within the text can additionally be used. The reason separators are now optional is because of the root brace extension. Once this extension is allowed, it no longer makes much sense to require separators as they would not 'feel' or look right for root level object properties.

# Property keys
Object property keys are extended such that they can be written as a string or as an unquoted sequence of characters. Duplicate or multiple instances of the same property key are accepted. If a property key is encountered that already exists during parsing, its value is simply replaced, not merged, with the new value. This plugs an underspecification hole within the json spec.

## Unquoted keys
Unquoted property keys have the following requirements.
1. Any codepoint except control characters, whitespace, line terminators, and the backslash character can be used.
2. Escape sequences are not supported.
3. The key cannot start with a '-' (U2D) or digit character
4. The key cannot contain any structural token
5. The key cannot be any of the keywords or keyword literals (case insensitive)

Unquoted keys start when the next non whitespace character is encountered and terminate when a subsequent whitespace character or a property key, value separator is encountered. The following regex example can be used to validate the key: **(?i)(^(null|true|false|infinity|nan)$|[{}\[\],])**.

# Root brace extension
In a super majority of json texts, the root value of a document is an object, thus requires initial braces surrounding the document. This also increases the indentation level. If a root object value essentially becomes a 'requirement' for all json documents, then the braces at the root should become implicit. In other words, we shouldn't have to add the root braces and a level of indentation for all of our documents.

jsonp documents are extended to support the root brace extension. This means that braces are optional and not required at the root of a document. It is now possible to write a document as a series of object properties at the root level. These properties are added to a root level object value that is implictly created by a parser.

A jsonp document can only contain a root value or a series of object properties that are added to an implicit root object value. A document cannot contain a mix of the two. It is possible to classify the document after the first or second token is encountered within the document.
