之前说到对请求行和请求头进行解析，获取到请求信息，现在我们有了请求信息，就要根据请求url映射到servlet进行处理，接下来开始看这个过程。tomcat处理请求的线程中`CoyoteAdapter.java`的`service`中,通过`postParseRequest(req, request, res, response);`方法根据请求url映射相关容器，映射容器的过程是在`connector.getService().getMapper().map(serverName, decodedURI, version, request.getMappingData());`中实现的。
+ 定位Context  
通过uri字符串和context的name字符串匹配，uri必须以context的name开头，且下一个字符为 '/'，则认为找到了context。
```java
private void internalMap(CharChunk host, CharChunk uri, String version, MappingData mappingData)
            throws IOException {
......
        MappedHost[] hosts = this.hosts;
        //忽略大小写匹配路径，找到 MappedHost
        MappedHost mappedHost = exactFindIgnoreCase(hosts, host);
        if (mappedHost == null) {
            // 没有找到的话，从第二个 "." 开始向后查找
            int firstDot = host.indexOf('.');
            if (firstDot > -1) {
                int offset = host.getOffset();
                try {
                    host.setOffset(firstDot + offset);
                    mappedHost = exactFindIgnoreCase(hosts, host);
                } finally {
                    // Make absolutely sure this gets reset
                    host.setOffset(offset);
                }
            }
            if (mappedHost == null) {
                mappedHost = defaultHost;
                if (mappedHost == null) {
                    return;
                }
            }
        }
        mappingData.host = mappedHost.object;

        if (uri.isNull()) {
            // Can't map context or wrapper without a uri
            return;
        }

        uri.setLimit(-1);

        // 映射Context
        ContextList contextList = mappedHost.contextList;
        MappedContext[] contexts = contextList.contexts;
        //到MappedContext[]中根据uri遍历，找到uri包含Context Name的Context
        int pos = find(contexts, uri);
        if (pos == -1) {
            return;
        }

        int lastSlash = -1;
        int uriEnd = uri.getEnd();
        int length = -1;
        boolean found = false;
        MappedContext context = null;
        while (pos >= 0) {
            context = contexts[pos];
            if (uri.startsWith(context.name)) {
                length = context.name.length();
                //uri长度和context的name长度相等，说明找到了对应的context，但是没有servlet
                if (uri.getLength() == length) {
                    found = true;
                    break;
                    //uri中从context的name的长度开始的字符串是否为/，是则找到
                    //例如 /MyXmlApp/myservlet 中找到一个 name为 /MyXmlApp 的context，并且name的下个字符串是 /
                    //意思是 uri必须包含context的name，并且下一个字符为  "/"
                } else if (uri.startsWithIgnoreCase("/", length)) {
                    found = true;
                    break;
                }
            }
            if (lastSlash == -1) {
                lastSlash = nthSlash(uri, contextList.nesting + 1);
            } else {
                lastSlash = lastSlash(uri);
            }
            uri.setEnd(lastSlash);
            pos = find(contexts, uri);
        }
        uri.setEnd(uriEnd);

        if (!found) {
            if (contexts[0].name.equals("")) {
                context = contexts[0];
            } else {
                context = null;
            }
        }
        if (context == null) {
            return;
        }

        mappingData.contextPath.setString(context.name);

        ContextVersion contextVersion = null;
        ContextVersion[] contextVersions = context.versions;
        final int versionCount = contextVersions.length;
        if (versionCount > 1) {
            Context[] contextObjects = new Context[contextVersions.length];
            for (int i = 0; i < contextObjects.length; i++) {
                contextObjects[i] = contextVersions[i].object;
            }
            mappingData.contexts = contextObjects;
            if (version != null) {
                contextVersion = exactFind(contextVersions, version);
            }
        }
        if (contextVersion == null) {
            // Return the latest version
            // The versions array is known to contain at least one element
            contextVersion = contextVersions[versionCount - 1];
        }
        mappingData.context = contextVersion.object;
        mappingData.contextSlashCount = contextVersion.slashCount;

        // 映射servlet，servlet是由Wrapper封装的
        if (!contextVersion.isPaused()) {
            internalMapWrapper(contextVersion, uri, mappingData);
        }

    }
```
+ 定位servlet  
定位servlet是一个很长的过程，由于平时我们大多都是精准路径匹配，所以我就看了前面一点，后面还有模糊匹配等方式没有深入去看了。过程其实很简单，在uri中把context对应的路径字符串去掉，后面的字符串就是servlet映射的请求url路径，根据此路径调用`internalMapExactWrapper`方法，到MappedWrapper[] wrappers数组中，匹配对应的MappedWrapper即可。
```java
    private void internalMapWrapper(ContextVersion contextVersion, CharChunk path, MappingData mappingData)
            throws IOException {

        int pathOffset = path.getOffset();
        int pathEnd = path.getEnd();
        boolean noServletPath = false;

        int length = contextVersion.path.length();
        //如果只有context路径，则无对应的servlet
        if (length == (pathEnd - pathOffset)) {
            noServletPath = true;
        }
        //servlet路径在context路径后面
        int servletPath = pathOffset + length;
        //截取context路径后面的字符串
        path.setOffset(servletPath);

        // Rule 1 -- Exact Match
        MappedWrapper[] exactWrappers = contextVersion.exactWrappers;
        internalMapExactWrapper(exactWrappers, path, mappingData);

        // Rule 2 -- Prefix Match
        boolean checkJspWelcomeFiles = false;
        MappedWrapper[] wildcardWrappers = contextVersion.wildcardWrappers;
        if (mappingData.wrapper == null) {
            internalMapWildcardWrapper(wildcardWrappers, contextVersion.nesting, path, mappingData);
            if (mappingData.wrapper != null && mappingData.jspWildCard) {
                char[] buf = path.getBuffer();
                if (buf[pathEnd - 1] == '/') {
                    /*
                     * Path ending in '/' was mapped to JSP servlet based on wildcard match (e.g., as specified in
                     * url-pattern of a jsp-property-group. Force the context's welcome files, which are interpreted as
                     * JSP files (since they match the url-pattern), to be considered. See Bugzilla 27664.
                     */
                    mappingData.wrapper = null;
                    checkJspWelcomeFiles = true;
                } else {
                    // See Bugzilla 27704
                    mappingData.wrapperPath.setChars(buf, path.getStart(), path.getLength());
                    mappingData.pathInfo.recycle();
                }
            }
        }

        if (mappingData.wrapper == null && noServletPath &&
                contextVersion.object.getMapperContextRootRedirectEnabled()) {
            // The path is empty, redirect to "/"
            path.append('/');
            pathEnd = path.getEnd();
            mappingData.redirectPath.setChars(path.getBuffer(), pathOffset, pathEnd - pathOffset);
            path.setEnd(pathEnd - 1);
            return;
        }

        // Rule 3 -- Extension Match
        MappedWrapper[] extensionWrappers = contextVersion.extensionWrappers;
        if (mappingData.wrapper == null && !checkJspWelcomeFiles) {
            internalMapExtensionWrapper(extensionWrappers, path, mappingData, true);
        }

        // Rule 4 -- Welcome resources processing for servlets
        if (mappingData.wrapper == null) {
            boolean checkWelcomeFiles = checkJspWelcomeFiles;
            if (!checkWelcomeFiles) {
                char[] buf = path.getBuffer();
                checkWelcomeFiles = (buf[pathEnd - 1] == '/');
            }
            if (checkWelcomeFiles) {
                for (int i = 0; (i < contextVersion.welcomeResources.length) && (mappingData.wrapper == null); i++) {
                    path.setOffset(pathOffset);
                    path.setEnd(pathEnd);
                    path.append(contextVersion.welcomeResources[i], 0, contextVersion.welcomeResources[i].length());
                    path.setOffset(servletPath);

                    // Rule 4a -- Welcome resources processing for exact macth
                    internalMapExactWrapper(exactWrappers, path, mappingData);

                    // Rule 4b -- Welcome resources processing for prefix match
                    if (mappingData.wrapper == null) {
                        internalMapWildcardWrapper(wildcardWrappers, contextVersion.nesting, path, mappingData);
                    }

                    // Rule 4c -- Welcome resources processing
                    // for physical folder
                    if (mappingData.wrapper == null && contextVersion.resources != null) {
                        String pathStr = path.toString();
                        WebResource file = contextVersion.resources.getResource(pathStr);
                        if (file != null && file.isFile()) {
                            internalMapExtensionWrapper(extensionWrappers, path, mappingData, true);
                            if (mappingData.wrapper == null && contextVersion.defaultWrapper != null) {
                                mappingData.wrapper = contextVersion.defaultWrapper.object;
                                mappingData.requestPath.setChars(path.getBuffer(), path.getStart(), path.getLength());
                                mappingData.wrapperPath.setChars(path.getBuffer(), path.getStart(), path.getLength());
                                mappingData.requestPath.setString(pathStr);
                                mappingData.wrapperPath.setString(pathStr);
                            }
                        }
                    }
                }

                path.setOffset(servletPath);
                path.setEnd(pathEnd);
            }

        }

        /*
         * welcome file processing - take 2 Now that we have looked for welcome files with a physical backing, now look
         * for an extension mapping listed but may not have a physical backing to it. This is for the case of index.jsf,
         * index.do, etc. A watered down version of rule 4
         */
        if (mappingData.wrapper == null) {
            boolean checkWelcomeFiles = checkJspWelcomeFiles;
            if (!checkWelcomeFiles) {
                char[] buf = path.getBuffer();
                checkWelcomeFiles = (buf[pathEnd - 1] == '/');
            }
            if (checkWelcomeFiles) {
                for (int i = 0; (i < contextVersion.welcomeResources.length) && (mappingData.wrapper == null); i++) {
                    path.setOffset(pathOffset);
                    path.setEnd(pathEnd);
                    path.append(contextVersion.welcomeResources[i], 0, contextVersion.welcomeResources[i].length());
                    path.setOffset(servletPath);
                    internalMapExtensionWrapper(extensionWrappers, path, mappingData, false);
                }

                path.setOffset(servletPath);
                path.setEnd(pathEnd);
            }
        }


        // Rule 7 -- Default servlet
        if (mappingData.wrapper == null && !checkJspWelcomeFiles) {
            if (contextVersion.defaultWrapper != null) {
                mappingData.wrapper = contextVersion.defaultWrapper.object;
                mappingData.requestPath.setChars(path.getBuffer(), path.getStart(), path.getLength());
                mappingData.wrapperPath.setChars(path.getBuffer(), path.getStart(), path.getLength());
                mappingData.matchType = MappingMatch.DEFAULT;
            }
            // Redirection to a folder
            char[] buf = path.getBuffer();
            if (contextVersion.resources != null && buf[pathEnd - 1] != '/') {
                String pathStr = path.toString();
                // Note: Check redirect first to save unnecessary getResource()
                // call. See BZ 62968.
                if (contextVersion.object.getMapperDirectoryRedirectEnabled()) {
                    WebResource file;
                    // Handle context root
                    if (pathStr.length() == 0) {
                        file = contextVersion.resources.getResource("/");
                    } else {
                        file = contextVersion.resources.getResource(pathStr);
                    }
                    if (file != null && file.isDirectory()) {
                        // Note: this mutates the path: do not do any processing
                        // after this (since we set the redirectPath, there
                        // shouldn't be any)
                        path.setOffset(pathOffset);
                        path.append('/');
                        mappingData.redirectPath.setChars(path.getBuffer(), path.getStart(), path.getLength());
                    } else {
                        mappingData.requestPath.setString(pathStr);
                        mappingData.wrapperPath.setString(pathStr);
                    }
                } else {
                    mappingData.requestPath.setString(pathStr);
                    mappingData.wrapperPath.setString(pathStr);
                }
            }
        }

        path.setOffset(pathOffset);
        path.setEnd(pathEnd);
    }

```
其实这样看，根据请求url映射servlet相关容器的过程并不复杂，本质就是字符串处理的过程。