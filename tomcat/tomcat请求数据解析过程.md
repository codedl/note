前面提到tomcat请求处理的io交互过程，现在开始看下经过io交互后tomcat是怎么处理请求数据的。首先到`AbstractProtocol.java`中的`process`方法，注意这个方法是在tomcat线程池分配的线程调用的。生成用来对请求字节流数据进行解析的Http11Processor。
```java
        public SocketState process(SocketWrapperBase<S> wrapper, SocketEvent status) {
            ......
            Processor processor = (Processor) wrapper.takeCurrentProcessor();
            ......
            try {
                ......
                if (processor == null) {
                    processor = recycledProcessors.pop();
                    ......
                }
                if (processor == null) {
                    processor = getProtocol().createProcessor();
                    register(processor);
                    ......
                }
                do {
                    state = processor.process(wrapper, status);
                    ......
                } while (state == SocketState.UPGRADING);
        }
//默认为 Http11Processor    
    protected Processor createProcessor() {
        Http11Processor processor = new Http11Processor(this, adapter);
        return processor;
    }
```
然后在生成的Http11Processor的`service`方法中对请求字节流数据进行正式的解析。
+ 请求行解析  
请求行格式如`GET / HTTP/1.1`，首先跳过空行，然后读取请求方式，再读取请求url，之后解析协议版本，逻辑很清晰。注意这里是在`fill(false)`方法中读取nio的ByteBuffer的数据。
```java
boolean parseRequestLine(boolean keptAlive, int connectionTimeout, int keepAliveTimeout) throws IOException {

        // check state
        if (!parsingRequestLine) {
            return true;
        }
        //
        // Skipping blank lines
        // 跳过空行
        if (parsingRequestLinePhase < 2) {
            do {
                // Read new bytes if needed
                if (byteBuffer.position() >= byteBuffer.limit()) {
                    if (keptAlive) {
                        // Haven't read any request data yet so use the keep-alive
                        // timeout.
                        wrapper.setReadTimeout(keepAliveTimeout);
                    }
                    //读取nio中ByteBuffer中数据
                    if (!fill(false)) {
                        // A read is pending, so no longer in initial state
                        parsingRequestLinePhase = 1;
                        return false;
                    }
                    // At least one byte of the request has been received.
                    // Switch to the socket timeout.
                    wrapper.setReadTimeout(connectionTimeout);
                }
                if (!keptAlive && byteBuffer.position() == 0 && byteBuffer.limit() >= CLIENT_PREFACE_START.length) {
                    boolean prefaceMatch = true;
                    for (int i = 0; i < CLIENT_PREFACE_START.length && prefaceMatch; i++) {
                        if (CLIENT_PREFACE_START[i] != byteBuffer.get(i)) {
                            prefaceMatch = false;
                        }
                    }
                    if (prefaceMatch) {
                        // HTTP/2 preface matched
                        parsingRequestLinePhase = -1;
                        return false;
                    }
                }
                // Set the start time once we start reading data (even if it is
                // just skipping blank lines)
                if (request.getStartTime() < 0) {
                    request.setStartTime(System.currentTimeMillis());
                }
                chr = byteBuffer.get();
            } while (chr == Constants.CR || chr == Constants.LF);
            byteBuffer.position(byteBuffer.position() - 1);

            parsingRequestLineStart = byteBuffer.position();
            parsingRequestLinePhase = 2;
        }
        //解析请求方式
        if (parsingRequestLinePhase == 2) {
            //
            // Reading the method name
            // Method name is a token
            //
            boolean space = false;
            while (!space) {
                // Read new bytes if needed
                if (byteBuffer.position() >= byteBuffer.limit()) {
                    if (!fill(false)) {
                        return false;
                    }
                }
                // Spec says method name is a token followed by a single SP but
                // also be tolerant of multiple SP and/or HT.
                int pos = byteBuffer.position();
                chr = byteBuffer.get();
                //http请求的开头字符为请求方式，以 空格(' ')或制表('\t')结尾
                if (chr == Constants.SP || chr == Constants.HT) {
                    space = true;
                    request.method().setBytes(byteBuffer.array(), parsingRequestLineStart,
                            pos - parsingRequestLineStart);
                } else if (!HttpParser.isToken(chr)) {
                    // Avoid unknown protocol triggering an additional error
                    request.protocol().setString(Constants.HTTP_11);
                    String invalidMethodValue = parseInvalid(parsingRequestLineStart, byteBuffer);
                    throw new IllegalArgumentException(sm.getString("iib.invalidmethod", invalidMethodValue));
                }
            }
            parsingRequestLinePhase = 3;
        }
        // 去除空字符
        if (parsingRequestLinePhase == 3) {
            // Spec says single SP but also be tolerant of multiple SP and/or HT
            boolean space = true;
            while (space) {
                // Read new bytes if needed
                if (byteBuffer.position() >= byteBuffer.limit()) {
                    if (!fill(false)) {
                        return false;
                    }
                }
                chr = byteBuffer.get();
                if (chr != Constants.SP && chr != Constants.HT) {
                    space = false;
                    byteBuffer.position(byteBuffer.position() - 1);
                }
            }
            parsingRequestLineStart = byteBuffer.position();
            parsingRequestLinePhase = 4;
        }
        //解析url
        if (parsingRequestLinePhase == 4) {
            // Mark the current buffer position

            int end = 0;
            //
            // Reading the URI
            //
            boolean space = false;
            while (!space) {
                // Read new bytes if needed
                if (byteBuffer.position() >= byteBuffer.limit()) {
                    if (!fill(false)) {
                        return false;
                    }
                }
                int pos = byteBuffer.position();
                prevChr = chr;
                chr = byteBuffer.get();
                if (prevChr == Constants.CR && chr != Constants.LF) {
                    // CR not followed by LF so not an HTTP/0.9 request and
                    // therefore invalid. Trigger error handling.
                    // Avoid unknown protocol triggering an additional error
                    request.protocol().setString(Constants.HTTP_11);
                    String invalidRequestTarget = parseInvalid(parsingRequestLineStart, byteBuffer);
                    throw new IllegalArgumentException(sm.getString("iib.invalidRequestTarget", invalidRequestTarget));
                }
                if (chr == Constants.SP || chr == Constants.HT) {
                    space = true;
                    end = pos;
                } else if (chr == Constants.CR) {
                    // HTTP/0.9 style request. CR is optional. LF is not.
                } else if (chr == Constants.LF) {
                    // HTTP/0.9 style request
                    // Stop this processing loop
                    space = true;
                    // Set blank protocol (indicates HTTP/0.9)
                    request.protocol().setString("");
                    // Skip the protocol processing
                    parsingRequestLinePhase = 7;
                    if (prevChr == Constants.CR) {
                        end = pos - 1;
                    } else {
                        end = pos;
                    }
                } else if (chr == Constants.QUESTION && parsingRequestLineQPos == -1) {
                    parsingRequestLineQPos = pos;
                } else if (parsingRequestLineQPos != -1 && !httpParser.isQueryRelaxed(chr)) {
                    // Avoid unknown protocol triggering an additional error
                    request.protocol().setString(Constants.HTTP_11);
                    // %nn decoding will be checked at the point of decoding
                    String invalidRequestTarget = parseInvalid(parsingRequestLineStart, byteBuffer);
                    throw new IllegalArgumentException(sm.getString("iib.invalidRequestTarget", invalidRequestTarget));
                } else if (httpParser.isNotRequestTargetRelaxed(chr)) {
                    // Avoid unknown protocol triggering an additional error
                    request.protocol().setString(Constants.HTTP_11);
                    // This is a general check that aims to catch problems early
                    // Detailed checking of each part of the request target will
                    // happen in Http11Processor#prepareRequest()
                    String invalidRequestTarget = parseInvalid(parsingRequestLineStart, byteBuffer);
                    throw new IllegalArgumentException(sm.getString("iib.invalidRequestTarget", invalidRequestTarget));
                }
            }
            if (parsingRequestLineQPos >= 0) {
                request.queryString().setBytes(byteBuffer.array(), parsingRequestLineQPos + 1,
                        end - parsingRequestLineQPos - 1);
                request.requestURI().setBytes(byteBuffer.array(), parsingRequestLineStart,
                        parsingRequestLineQPos - parsingRequestLineStart);
            } else {
                //把解析到的url记下来
                request.requestURI().setBytes(byteBuffer.array(), parsingRequestLineStart,
                        end - parsingRequestLineStart);
            }
            // HTTP/0.9 processing jumps to stage 7.
            // Don't want to overwrite that here.
            if (parsingRequestLinePhase == 4) {
                parsingRequestLinePhase = 5;
            }
        }
        if (parsingRequestLinePhase == 5) {
            // Spec says single SP but also be tolerant of multiple and/or HT
            boolean space = true;
            while (space) {
                // Read new bytes if needed
                if (byteBuffer.position() >= byteBuffer.limit()) {
                    if (!fill(false)) {
                        return false;
                    }
                }
                byte chr = byteBuffer.get();
                if (chr != Constants.SP && chr != Constants.HT) {
                    space = false;
                    byteBuffer.position(byteBuffer.position() - 1);
                }
            }
            parsingRequestLineStart = byteBuffer.position();
            parsingRequestLinePhase = 6;

            // Mark the current buffer position
            end = 0;
        }
        //协议版本
        if (parsingRequestLinePhase == 6) {
            //
            // Reading the protocol
            // Protocol is always "HTTP/" DIGIT "." DIGIT
            //
            while (!parsingRequestLineEol) {
                // Read new bytes if needed
                if (byteBuffer.position() >= byteBuffer.limit()) {
                    if (!fill(false)) {
                        return false;
                    }
                }

                int pos = byteBuffer.position();
                prevChr = chr;
                chr = byteBuffer.get();
                if (chr == Constants.CR) {
                    // Possible end of request line. Need LF next else invalid.
                } else if (prevChr == Constants.CR && chr == Constants.LF) {
                    // CRLF is the standard line terminator
                    end = pos - 1;
                    parsingRequestLineEol = true;
                } else if (chr == Constants.LF) {
                    // LF is an optional line terminator
                    end = pos;
                    parsingRequestLineEol = true;
                } else if (prevChr == Constants.CR || !HttpParser.isHttpProtocol(chr)) {
                    String invalidProtocol = parseInvalid(parsingRequestLineStart, byteBuffer);
                    throw new IllegalArgumentException(sm.getString("iib.invalidHttpProtocol", invalidProtocol));
                }
            }

            if (end - parsingRequestLineStart > 0) {
                request.protocol().setBytes(byteBuffer.array(), parsingRequestLineStart, end - parsingRequestLineStart);
                parsingRequestLinePhase = 7;
            }
            // If no protocol is found, the ISE below will be triggered.
        }
        if (parsingRequestLinePhase == 7) {
            // Parsing is complete. Return and clean-up.
            parsingRequestLine = false;
            parsingRequestLinePhase = 0;
            parsingRequestLineEol = false;
            parsingRequestLineStart = 0;
            return true;
        }
        throw new IllegalStateException(sm.getString("iib.invalidPhase", Integer.valueOf(parsingRequestLinePhase)));
    }
```
+ 请求头解析  
http请求头格式如`name:value`，解析先按行读取字符串，读取到":"则认为读到一个请求头，使用`MimeHeaderField`封装请求头字段，并返回当前请求头的value值到`headerData.headerValue`，再继续读取字节，读到"\r"为止，然后转换成字符串`headerData.headerValue.setBytes(byteBuffer.array(), headerData.start,headerData.lastSignificantChar - headerData.start);`保存到headerData.headerValue引用指向的MimeHeaderField对象。
```java

private HeaderParseStatus parseHeader() throws IOException {


            // Read new bytes if needed
            if (byteBuffer.position() >= byteBuffer.limit()) {
                if (!fill(false)) {
                    return HeaderParseStatus.NEED_MORE_DATA;
                }
            }

            prevChr = chr;
            chr = byteBuffer.get();

            if (chr == Constants.CR && prevChr != Constants.CR) {
                // Possible start of CRLF - process the next byte.
            } else if (chr == Constants.LF) {
                // CRLF or LF is an acceptable line terminator
                return HeaderParseStatus.DONE;
            } else {
                if (prevChr == Constants.CR) {
                    // Must have read two bytes (first was CR, second was not LF)
                    byteBuffer.position(byteBuffer.position() - 2);
                } else {
                    // Must have only read one byte
                    byteBuffer.position(byteBuffer.position() - 1);
                }
                break;
            }
        }

        if (headerParsePos == HeaderParsePosition.HEADER_START) {
            // Mark the current buffer position
            headerData.start = byteBuffer.position();
            headerData.lineStart = headerData.start;
            headerParsePos = HeaderParsePosition.HEADER_NAME;
        }

        //
        // Reading the header name
        // Header name is always US-ASCII
        //

        while (headerParsePos == HeaderParsePosition.HEADER_NAME) {

            // Read new bytes if needed
            if (byteBuffer.position() >= byteBuffer.limit()) {
                if (!fill(false)) { // parse header
                    return HeaderParseStatus.NEED_MORE_DATA;
                }
            }

            int pos = byteBuffer.position();
            chr = byteBuffer.get();
            if (chr == Constants.COLON) {
                if (headerData.start == pos) {
                    // Zero length header name - not valid.
                    // skipLine() will handle the error
                    return skipLine(false);
                }
                headerParsePos = HeaderParsePosition.HEADER_VALUE_START;
                headerData.headerValue = headers.addValue(byteBuffer.array(), headerData.start, pos - headerData.start);
                pos = byteBuffer.position();
                // Mark the current buffer position
                headerData.start = pos;
                headerData.realPos = pos;
                headerData.lastSignificantChar = pos;
                break;
            } else if (!HttpParser.isToken(chr)) {
                // Non-token characters are illegal in header names
                // Parsing continues so the error can be reported in context
                headerData.lastSignificantChar = pos;
                byteBuffer.position(byteBuffer.position() - 1);
                // skipLine() will handle the error
                return skipLine(false);
            }

            // chr is next byte of header name. Convert to lowercase.
            if (chr >= Constants.A && chr <= Constants.Z) {
                byteBuffer.put(pos, (byte) (chr - Constants.LC_OFFSET));
            }
        }

        // Skip the line and ignore the header
        if (headerParsePos == HeaderParsePosition.HEADER_SKIPLINE) {
            return skipLine(false);
        }

        //
        // Reading the header value (which can be spanned over multiple lines)
        //

        while (headerParsePos == HeaderParsePosition.HEADER_VALUE_START ||
                headerParsePos == HeaderParsePosition.HEADER_VALUE ||
                headerParsePos == HeaderParsePosition.HEADER_MULTI_LINE) {

            if (headerParsePos == HeaderParsePosition.HEADER_VALUE_START) {
                // Skipping spaces
                while (true) {
                    // Read new bytes if needed
                    if (byteBuffer.position() >= byteBuffer.limit()) {
                        if (!fill(false)) {// parse header
                            // HEADER_VALUE_START
                            return HeaderParseStatus.NEED_MORE_DATA;
                        }
                    }

                    chr = byteBuffer.get();
                    if (chr != Constants.SP && chr != Constants.HT) {
                        headerParsePos = HeaderParsePosition.HEADER_VALUE;
                        byteBuffer.position(byteBuffer.position() - 1);
                        // Avoids prevChr = chr at start of header value
                        // parsing which causes problems when chr is CR
                        // (in the case of an empty header value)
                        chr = 0;
                        break;
                    }
                }
            }
            if (headerParsePos == HeaderParsePosition.HEADER_VALUE) {

                // Reading bytes until the end of the line
                boolean eol = false;
                while (!eol) {

                    // Read new bytes if needed
                    if (byteBuffer.position() >= byteBuffer.limit()) {
                        if (!fill(false)) {// parse header
                            // HEADER_VALUE
                            return HeaderParseStatus.NEED_MORE_DATA;
                        }
                    }

                    prevChr = chr;
                    chr = byteBuffer.get();
                    if (chr == Constants.CR && prevChr != Constants.CR) {
                        // CR is only permitted at the start of a CRLF sequence.
                        // Possible start of CRLF - process the next byte.
                    } else if (chr == Constants.LF) {
                        // CRLF or LF is an acceptable line terminator
                        eol = true;
                    } else if (prevChr == Constants.CR) {
                        // Invalid value - also need to delete header
                        return skipLine(true);
                    } else if (HttpParser.isControl(chr) && chr != Constants.HT) {
                        // Invalid value - also need to delete header
                        return skipLine(true);
                    } else if (chr == Constants.SP || chr == Constants.HT) {
                        byteBuffer.put(headerData.realPos, chr);
                        headerData.realPos++;
                    } else {
                        byteBuffer.put(headerData.realPos, chr);
                        headerData.realPos++;
                        headerData.lastSignificantChar = headerData.realPos;
                    }
                }

                // Ignore whitespaces at the end of the line
                headerData.realPos = headerData.lastSignificantChar;

                // Checking the first character of the new line. If the character
                // is a LWS, then it's a multiline header
                headerParsePos = HeaderParsePosition.HEADER_MULTI_LINE;
            }
            // Read new bytes if needed
            if (byteBuffer.position() >= byteBuffer.limit()) {
                if (!fill(false)) {// parse header
                    // HEADER_MULTI_LINE
                    return HeaderParseStatus.NEED_MORE_DATA;
                }
            }

            byte peek = byteBuffer.get(byteBuffer.position());
            if (headerParsePos == HeaderParsePosition.HEADER_MULTI_LINE) {
                if (peek != Constants.SP && peek != Constants.HT) {
                    headerParsePos = HeaderParsePosition.HEADER_START;
                    break;
                } else {
                    // Copying one extra space in the buffer (since there must
                    // be at least one space inserted between the lines)
                    byteBuffer.put(headerData.realPos, peek);
                    headerData.realPos++;
                    headerParsePos = HeaderParsePosition.HEADER_VALUE_START;
                }
            }
        }
        // Set the header value
        headerData.headerValue.setBytes(byteBuffer.array(), headerData.start,
                headerData.lastSignificantChar - headerData.start);
        headerData.recycle();
        return HeaderParseStatus.HAVE_MORE_HEADERS;
    }
```    
+ 请求体解析  
请求体获取方式如下，从请求request中拿到BufferedReader,然后调用`readLine`就行。
```java
        BufferedReader reader = request.getReader();
        String line;
        while ((line=reader.readLine())!=null){
            System.out.println(line);
        }
```
tomcat会用CoyoteReader去封装内置的inputBuffer，生成BufferedReader。
```java
    public BufferedReader getReader() throws IOException {

        if (usingInputStream) {
            throw new IllegalStateException(sm.getString("coyoteRequest.getReader.ise"));
        }

        if (coyoteRequest.getCharacterEncoding() == null) {
            Context context = getContext();
            if (context != null) {
                String enc = context.getRequestCharacterEncoding();
                if (enc != null) {
                    setCharacterEncoding(enc);
                }
            }
        }

        usingReader = true;

        inputBuffer.checkConverter();
        if (reader == null) {
            reader = new CoyoteReader(inputBuffer);
        }
        return reader;
    }
```
其实也很简答，就是到内置的缓冲区中读取数据，然后使用编码成字符串即可。
```java
    public String readLine() throws IOException {

        if (lineBuffer == null) {
            lineBuffer = new char[MAX_LINE_LENGTH];
        }

        String result = null;

        int pos = 0;
        int end = -1;
        int skip = -1;
        StringBuilder aggregator = null;
        while (end < 0) {
            mark(MAX_LINE_LENGTH);
            while ((pos < MAX_LINE_LENGTH) && (end < 0)) {
                int nRead = read(lineBuffer, pos, MAX_LINE_LENGTH - pos);
                if (nRead < 0) {
                    if (pos == 0 && aggregator == null) {
                        return null;
                    }
                    end = pos;
                    skip = pos;
                }
                for (int i = pos; (i < (pos + nRead)) && (end < 0); i++) {
                    if (lineBuffer[i] == LINE_SEP[0]) {
                        end = i;
                        skip = i + 1;
                        char nextchar;
                        if (i == (pos + nRead - 1)) {
                            nextchar = (char) read();
                        } else {
                            nextchar = lineBuffer[i + 1];
                        }
                        if (nextchar == LINE_SEP[1]) {
                            skip++;
                        }
                    } else if (lineBuffer[i] == LINE_SEP[1]) {
                        end = i;
                        skip = i + 1;
                    }
                }
                if (nRead > 0) {
                    pos += nRead;
                }
            }
            if (end < 0) {
                if (aggregator == null) {
                    aggregator = new StringBuilder();
                }
                aggregator.append(lineBuffer);
                pos = 0;
            } else {
                reset();
                // No need to check return value. We know there are at least skip characters available.
                skip(skip);
            }
        }

        if (aggregator == null) {
            result = new String(lineBuffer, 0, end);
        } else {
            aggregator.append(lineBuffer, 0, end);
            result = aggregator.toString();
        }

        return result;

    }
```