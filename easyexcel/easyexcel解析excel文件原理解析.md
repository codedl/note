easyexcel是对poi基于事件机制解析excel文件的封装，poi又对sax解析xml文件进行了封装，所以要理解easyexcel解析的过程，就要先从sax开始。
+ sax解析xml文件  
使用sax解析xml文件时只要注册ContentHandler就行，sax是按行解析的，解析时会调用ContentHandler接口的钩子函数。
```java
        SAXParserFactory factory = SAXParserFactory.newInstance();
        SAXParser saxParser = factory.newSAXParser();
        XMLReader xmlReader = saxParser.getXMLReader();
        xmlReader.setContentHandler(new DefaultHandler(){
            @Override
            public void startElement(String uri, String localName, String qName, Attributes attributes){
                log.info("startElement-->>uri:{},localName:{},qName:{}",uri, localName,qName);
            }
            @Override
            public void endElement(String uri, String localName, String qName){
                log.info("endElement-->>uri:{},localName:{},qName:{}",uri, localName,qName);
            }
            @Override
            public void characters(char[] ch, int start, int length){
                log.info("characters-->>ch:{}, start:{}, length:{}", new String(ch,start,length), start, length);
            }
        });
        InputStream inputStream = new FileInputStream(Var.xmlFile);
        xmlReader.parse(new InputSource(inputStream));
        inputStream.close();
```
+ poi解析excel文件  
poi基于事件解析excel文件是对sax技术的封装，打开excel文件后，获取到所有sheet页，然后再类似xml逐行解析。
```java
        OPCPackage opcPackage = OPCPackage.open(Var.excelFile, PackageAccess.READ);
        XSSFReader reader = new XSSFReader(opcPackage);
        XMLReader xmlReader = XMLReaderFactory.createXMLReader();
        xmlReader.setContentHandler(new SheetHandler());
        Iterator<InputStream> sheetsData = reader.getSheetsData();
        while (sheetsData.hasNext()){
            InputStream next = sheetsData.next();
            InputSource inputSource = new InputSource(next);
            xmlReader.parse(inputSource);
            next.close();
        }
```
在poi解析大excel文件的基础上出现了easyexcel，先看下easyexcel的用法，真的很easy啊，调静态方法read，传文件路径就行，获取到每行数据时，会帮我们封装成实体类，我们在invoke中进行处理即可。
```java
EasyExcel.read(Var.excelFile, WfqData.class,
                    new AnalysisEventListener<WfqData>() {
                        @Override
                        public void invoke(WfqData data, AnalysisContext analysisContext) {
                            // 按行读文件
                        }

                        @Override
                        public void doAfterAllAnalysed(AnalysisContext analysisContext) {
                            log.info("处理文件结束");
                        }
                    }).headRowNumber(2)// 表头第二行，第一行是免责申明
                    .doReadAll();
```
接下来开始看easyexcel的源码。`read()`方法中会构造`ReadWorkbook`对象，然后设置excel文件的File对象，注册读监听器。
```java
    public static ExcelReaderBuilder read(String pathName, Class head, ReadListener readListener) {
        ExcelReaderBuilder excelReaderBuilder = new ExcelReaderBuilder();
        excelReaderBuilder.file(pathName);
        if (head != null) {
            excelReaderBuilder.head(head);
        }
        if (readListener != null) {
            excelReaderBuilder.registerReadListener(readListener);
        }
        return excelReaderBuilder;
    }
```
然后构造`ExcelReader`对象用来读取excel文件。
```java
    public void doReadAll() {
        ExcelReader excelReader = build();
        excelReader.readAll();
        excelReader.finish();
    }
```
接下来根据excel文件类型确定解析器，
```java
private void choiceExcelExecutor(ReadWorkbook readWorkbook) throws Exception {
        ExcelTypeEnum excelType = ExcelTypeEnum.valueOf(readWorkbook);
        switch (excelType) {
            case XLS:
                ......
                break;
            case XLSX:
                XlsxReadContext xlsxReadContext = new DefaultXlsxReadContext(readWorkbook, ExcelTypeEnum.XLSX);
                analysisContext = xlsxReadContext;
                excelReadExecutor = new XlsxSaxAnalyser(xlsxReadContext, null);
                break;
            default:
                break;
        }
    }
```
然后设置一堆的配置，在解析的时候会用到，这些配置都保存在`DefaultXlsxReadContext`中，使用默认的解析事件处理器`DefaultAnalysisEventProcessor`，在解析excel的过程中会通过这个事件处理器调用我们注册进来的监听器方法。
```java
    public AnalysisContextImpl(ReadWorkbook readWorkbook, ExcelTypeEnum actualExcelType) {
        if (readWorkbook == null) {
            throw new IllegalArgumentException("Workbook argument cannot be null");
        }
        switch (actualExcelType) {
            case XLS:
                readWorkbookHolder = new XlsReadWorkbookHolder(readWorkbook);
                break;
            case XLSX:
                readWorkbookHolder = new XlsxReadWorkbookHolder(readWorkbook);
                break;
            default:
                break;
        }
        currentReadHolder = readWorkbookHolder;
        analysisEventProcessor = new DefaultAnalysisEventProcessor();
        if (LOGGER.isDebugEnabled()) {
            LOGGER.debug("Initialization 'AnalysisContextImpl' complete");
        }
    }
```
解析前最重要的一环就是打开excel文档`OPCPackage.open(xlsxReadWorkbookHolder.getFile());`，这里就是我们熟悉的poi解析excel文件做的事情，然后获取每个要解析的sheet页，使用ReadSheet包装。
```java
    public XlsxSaxAnalyser(XlsxReadContext xlsxReadContext, InputStream decryptedStream) throws Exception {
        this.xlsxReadContext = xlsxReadContext;
        // Initialize cache
        XlsxReadWorkbookHolder xlsxReadWorkbookHolder = xlsxReadContext.xlsxReadWorkbookHolder();

        OPCPackage pkg = readOpcPackage(xlsxReadWorkbookHolder, decryptedStream);
        xlsxReadWorkbookHolder.setOpcPackage(pkg);

        ArrayList<PackagePart> packageParts = pkg.getPartsByContentType(XSSFRelation.SHARED_STRINGS.getContentType());

        if (!CollectionUtils.isEmpty(packageParts)) {
            PackagePart sharedStringsTablePackagePart = packageParts.get(0);

            // Specify default cache
            defaultReadCache(xlsxReadWorkbookHolder, sharedStringsTablePackagePart);

            // Analysis sharedStringsTable.xml
            analysisSharedStringsTable(sharedStringsTablePackagePart.getInputStream(), xlsxReadWorkbookHolder);
        }

        XSSFReader xssfReader = new XSSFReader(pkg);
        analysisUse1904WindowDate(xssfReader, xlsxReadWorkbookHolder);

        xlsxReadWorkbookHolder.setStylesTable(xssfReader.getStylesTable());
        sheetList = new ArrayList<ReadSheet>();
        sheetMap = new HashMap<Integer, InputStream>();
        commentsTableMap = new HashMap<Integer, CommentsTable>();
        XSSFReader.SheetIterator ite = (XSSFReader.SheetIterator)xssfReader.getSheetsData();
        int index = 0;
        if (!ite.hasNext()) {
            throw new ExcelAnalysisException("Can not find any sheet!");
        }
        while (ite.hasNext()) {
            InputStream inputStream = ite.next();
            sheetList.add(new ReadSheet(index, ite.getSheetName()));
            sheetMap.put(index, inputStream);
            if (xlsxReadContext.readWorkbookHolder().getExtraReadSet().contains(CellExtraTypeEnum.COMMENT)) {
                CommentsTable commentsTable = ite.getSheetComments();
                if (null != commentsTable) {
                    commentsTableMap.put(index, commentsTable);
                }
            }
            index++;
        }
    }
```
解析是在`XlsxSaxAnalyser#execute`方法中进行的，依次遍历获取的每个sheet页，保存下来后进行解析，就是poi解析excel文件的过程了。
```java
    public void execute() {
        for (ReadSheet readSheet : sheetList) {
            readSheet = SheetUtils.match(readSheet, xlsxReadContext);
            if (readSheet != null) {
                xlsxReadContext.currentSheet(readSheet);
                parseXmlSource(sheetMap.get(readSheet.getSheetNo()), new XlsxRowHandler(xlsxReadContext));
                // Read comments
                readComments(readSheet);
                // The last sheet is read
                xlsxReadContext.analysisEventProcessor().endSheet(xlsxReadContext);
            }
        }
    }

        private void parseXmlSource(InputStream inputStream, ContentHandler handler) {
        InputSource inputSource = new InputSource(inputStream);
        try {
            SAXParserFactory saxFactory;
            String xlsxSAXParserFactoryName = xlsxReadContext.xlsxReadWorkbookHolder().getSaxParserFactoryName();
            if (StringUtils.isEmpty(xlsxSAXParserFactoryName)) {
                saxFactory = SAXParserFactory.newInstance();
            } else {
                saxFactory = SAXParserFactory.newInstance(xlsxSAXParserFactoryName, null);
            }
            ...
            SAXParser saxParser = saxFactory.newSAXParser();
            XMLReader xmlReader = saxParser.getXMLReader();
            xmlReader.setContentHandler(handler);
            xmlReader.parse(inputSource);
            inputStream.close();
        }
    }
```
每次解析时都会回调`XlsxRowHandler`中的`startElement、characters、endElement`方法。  
`RowTagHandler#startElement()`方法中会先获取当前要解析的行，然后与之前解析到的行进行对比，防止遗漏。
```java
    public void startElement(XlsxReadContext xlsxReadContext, String name, Attributes attributes) {
        XlsxReadSheetHolder xlsxReadSheetHolder = xlsxReadContext.xlsxReadSheetHolder();
        int rowIndex = PositionUtils.getRowByRowTagt(attributes.getValue(ExcelXmlConstants.ATTRIBUTE_R),
            xlsxReadSheetHolder.getRowIndex());
        Integer lastRowIndex = xlsxReadContext.readSheetHolder().getRowIndex();
        while (lastRowIndex + 1 < rowIndex) {
            xlsxReadContext.readRowHolder(new ReadRowHolder(lastRowIndex + 1, RowTypeEnum.EMPTY,
                xlsxReadSheetHolder.getGlobalConfiguration(), new LinkedHashMap<Integer, Cell>()));
            xlsxReadContext.analysisEventProcessor().endRow(xlsxReadContext);
            xlsxReadSheetHolder.setColumnIndex(null);
            xlsxReadSheetHolder.setCellMap(new LinkedHashMap<Integer, Cell>());
            lastRowIndex++;
        }
        xlsxReadSheetHolder.setRowIndex(rowIndex);
    }
```
解析完一行后，在`AbstractCellValueTagHandler#endElement`中会将当前行的列数和对应的值保存到`cellMap(private Map<Integer, Cell> cellMap;)`中。
```java
public void endElement(XlsxReadContext xlsxReadContext, String name) {
        XlsxReadSheetHolder xlsxReadSheetHolder = xlsxReadContext.xlsxReadSheetHolder();
        CellData tempCellData = xlsxReadSheetHolder.getTempCellData();
        StringBuilder tempData = xlsxReadSheetHolder.getTempData();
        CellDataTypeEnum oldType = tempCellData.getType();
        switch (oldType) {
            case DIRECT_STRING:
            case STRING:
            case ERROR:
                tempCellData.setStringValue(tempData.toString());
                break;
            case BOOLEAN:
                tempCellData.setBooleanValue(BooleanUtils.valueOf(tempData.toString()));
                break;
            case NUMBER:
            case EMPTY:
                tempCellData.setType(CellDataTypeEnum.NUMBER);
                tempCellData.setNumberValue(new BigDecimal(tempData.toString()));
                break;
            default:
                throw new IllegalStateException("Cannot set values now");
        }

        // set string value
        setStringValue(xlsxReadContext);

        if (tempCellData.getStringValue() != null
            && xlsxReadContext.currentReadHolder().globalConfiguration().getAutoTrim()) {
            tempCellData.setStringValue(tempCellData.getStringValue());
        }

        tempCellData.checkEmpty();
        xlsxReadSheetHolder.getCellMap().put(xlsxReadSheetHolder.getColumnIndex(), tempCellData);
    }
```
然后在`RowTagHandler#endElement`中调用`xlsxReadContext.analysisEventProcessor().endRow(xlsxReadContext);`方法进行处理，这里会调用`DefaultAnalysisEventProcessor#dealData`处理每行的实际数据，其实就是调用读监听器的invoke方法，也就是我们在`EasyExcel.read`中传的第三个参数。
```java
    private void dealData(AnalysisContext analysisContext) {
        ReadRowHolder readRowHolder = analysisContext.readRowHolder();
        Map<Integer, CellData> cellDataMap = (Map)readRowHolder.getCellMap();
        readRowHolder.setCurrentRowAnalysisResult(cellDataMap);
        int rowIndex = readRowHolder.getRowIndex();
        int currentHeadRowNumber = analysisContext.readSheetHolder().getHeadRowNumber();

        boolean isData = rowIndex >= currentHeadRowNumber;

        // Last head column
        if (!isData && currentHeadRowNumber == rowIndex + 1) {
            buildHead(analysisContext, cellDataMap);
        }
        // Now is data
        for (ReadListener readListener : analysisContext.currentReadHolder().readListenerList()) {
            try {
                if (isData) {
                    readListener.invoke(readRowHolder.getCurrentRowAnalysisResult(), analysisContext);
                } else {
                    readListener.invokeHead(cellDataMap, analysisContext);
                }
            } catch (Exception e) {
                onException(analysisContext, e);
                break;
            }
            if (!readListener.hasNext(analysisContext)) {
                throw new ExcelAnalysisStopException();
            }
        }
    }
```
完成一个sheet页的解析，就会调用每个监听的doAfterAllAnalysed()方法。
```java
    public void endSheet(AnalysisContext analysisContext) {
        for (ReadListener readListener : analysisContext.currentReadHolder().readListenerList()) {
            readListener.doAfterAllAnalysed(analysisContext);
        }
    }
```
至此完成了对excel文件的解析。
**总结下，easyexcel解析excel文件，先初始化用来解析excel文件用到的参数，比如文件路径、表头行、行数据class等，然后使用poi基于事件的机制进行解析。获取到数据后会把列索引和cell数据保存到map，然后调用每个监听器的invoke方法进行处理。有不对的地方请大神指出，欢迎大家一起讨论交流，共同进步。**
