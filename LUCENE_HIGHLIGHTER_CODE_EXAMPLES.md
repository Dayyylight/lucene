# Lucene Highlighter 代码示例与实践指南

## 实际代码示例

### 1. Unified Highlighter 基础使用

```java
// 基本设置和使用示例
public class UnifiedHighlighterExample {
    
    public void basicHighlighting() throws IOException {
        // 创建搜索器和分析器
        IndexSearcher searcher = new IndexSearcher(indexReader);
        Analyzer analyzer = new StandardAnalyzer();
        
        // 创建统一高亮器
        UnifiedHighlighter highlighter = new UnifiedHighlighter(searcher, analyzer);
        
        // 设置参数
        highlighter.setMaxLength(10000);  // 最大分析字符数
        highlighter.setMaxNoHighlightPassages(1);  // 无匹配时返回的片段数
        
        // 执行查询
        Query query = new QueryParser("content", analyzer).parse("java programming");
        TopDocs topDocs = searcher.search(query, 10);
        
        // 生成高亮片段
        String[] fragments = highlighter.highlight("content", query, topDocs);
        
        // 输出结果
        for (int i = 0; i < fragments.length; i++) {
            System.out.println("Doc " + i + ": " + fragments[i]);
        }
    }
}
```

### 2. 自定义配置的高级示例

```java
public class AdvancedHighlighterExample {
    
    public void advancedConfiguration() throws IOException {
        IndexSearcher searcher = new IndexSearcher(indexReader);
        Analyzer analyzer = new StandardAnalyzer();
        
        // 创建自定义配置的高亮器
        UnifiedHighlighter highlighter = new UnifiedHighlighter(searcher, analyzer) {
            
            // 自定义片段分割策略
            @Override
            protected BreakIterator getBreakIterator(String field) {
                // 使用长度目标分割器，目标长度150字符
                return new LengthGoalBreakIterator(
                    BreakIterator.getSentenceInstance(Locale.ROOT), 150);
            }
            
            // 自定义评分器
            @Override
            protected PassageScorer getScorer(String field) {
                // 使用自定义参数的BM25评分器
                return new PassageScorer(1.5f, 0.8f, 100f);
            }
            
            // 自定义格式化器
            @Override
            protected PassageFormatter getFormatter(String field) {
                return new DefaultPassageFormatter(
                    "<mark>", "</mark>",    // 使用HTML mark标签
                    " ... ",                // 自定义省略号
                    true                    // 开启HTML转义
                );
            }
            
            // 自定义片段排序
            @Override
            protected Comparator<Passage> getPassageSortComparator(String field) {
                // 按评分降序，然后按位置升序
                return Comparator.comparing(Passage::getScore).reversed()
                    .thenComparing(Passage::getStartOffset);
            }
        };
        
        // 设置最大片段数
        highlighter.setMaxLength(50000);
        
        Query query = new TermQuery(new Term("content", "lucene"));
        TopDocs docs = searcher.search(query, 5);
        
        // 生成多个字段的高亮
        String[] fields = {"title", "content", "summary"};
        Map<String, String[]> highlights = highlighter.highlightFields(fields, query, docs);
        
        for (String field : fields) {
            System.out.println("=== " + field + " ===");
            String[] fieldHighlights = highlights.get(field);
            if (fieldHighlights != null) {
                for (int i = 0; i < fieldHighlights.length; i++) {
                    System.out.println("Doc " + i + ": " + fieldHighlights[i]);
                }
            }
        }
    }
}
```

### 3. Fast Vector Highlighter 示例

```java
public class VectorHighlighterExample {
    
    public void vectorHighlighting() throws IOException {
        IndexReader reader = DirectoryReader.open(directory);
        
        // 创建快速向量高亮器
        FastVectorHighlighter highlighter = new FastVectorHighlighter(
            true,   // 启用短语高亮
            true    // 启用字段匹配
        );
        
        // 创建字段查询
        Query query = new QueryParser("content", new StandardAnalyzer())
            .parse("\"Apache Lucene\" OR \"search engine\"");
        FieldQuery fieldQuery = highlighter.getFieldQuery(query, reader);
        
        // 设置高亮标签
        String[] preTags = {"<em class='highlight'>"};
        String[] postTags = {"</em>"};
        
        // 为每个文档生成高亮片段
        for (int docId = 0; docId < 10; docId++) {
            String[] fragments = highlighter.getBestFragments(
                fieldQuery, reader, docId, "content",
                200,    // 片段长度
                3,      // 最大片段数
                preTags, postTags
            );
            
            System.out.println("Document " + docId + " fragments:");
            for (String fragment : fragments) {
                System.out.println("  " + fragment);
            }
        }
    }
}
```

### 4. Classic Highlighter 示例

```java
public class ClassicHighlighterExample {
    
    public void classicHighlighting() throws IOException, InvalidTokenOffsetsException {
        Analyzer analyzer = new StandardAnalyzer();
        
        // 创建查询评分器
        Query query = new QueryParser("content", analyzer).parse("machine learning");
        QueryScorer scorer = new QueryScorer(query);
        
        // 创建高亮器
        Highlighter highlighter = new Highlighter(
            new SimpleHTMLFormatter("<b>", "</b>"),  // 格式化器
            new DefaultEncoder(),                     // 编码器
            scorer                                   // 评分器
        );
        
        // 设置片段分割器
        highlighter.setTextFragmenter(new SimpleSpanFragmenter(scorer, 100));
        
        // 文档文本
        String text = "Machine learning is a powerful tool for data analysis...";
        
        // 生成高亮片段
        TokenStream tokenStream = analyzer.tokenStream("content", text);
        String[] fragments = highlighter.getBestFragments(tokenStream, text, 3);
        
        for (String fragment : fragments) {
            System.out.println(fragment);
        }
    }
}
```

## 关键源码解析

### 1. UnifiedHighlighter.highlightFields() 核心流程

```java
// 位置：org.apache.lucene.search.uhighlight.UnifiedHighlighter
public Map<String, String[]> highlightFields(String[] fieldsIn, Query query, TopDocs topDocs, int[] maxPassagesIn) throws IOException {
    
    // 1. 输入验证和预处理
    Map<String, String[]> resultMap = new HashMap<>();
    if (fieldsIn.length == 0 || topDocs.scoreDocs.length == 0) {
        return resultMap;
    }
    
    // 2. 字段分组和策略选择
    Map<String, Set<HighlightFlag>> fieldFlags = new HashMap<>();
    Map<String, FieldHighlighter> fieldHighlighters = new HashMap<>();
    
    for (String field : fieldsIn) {
        // 为每个字段创建专门的高亮器
        FieldHighlighter fieldHighlighter = new FieldHighlighter(
            field,
            getFieldOffsetStrategy(field, query),
            getBreakIterator(field),
            getScorer(field),
            maxPassagesIn[0],
            getMaxNoHighlightPassages(field),
            getFormatter(field),
            getPassageSortComparator(field)
        );
        fieldHighlighters.put(field, fieldHighlighter);
    }
    
    // 3. 批量处理文档
    for (ScoreDoc scoreDoc : topDocs.scoreDocs) {
        LeafReaderContext leafReaderContext = reader.leaves().get(ReaderUtil.subIndex(scoreDoc.doc, reader.leaves()));
        LeafReader leafReader = leafReaderContext.reader();
        int docId = scoreDoc.doc - leafReaderContext.docBase;
        
        // 4. 为每个字段生成高亮
        for (String field : fieldsIn) {
            FieldHighlighter fieldHighlighter = fieldHighlighters.get(field);
            
            // 获取文档内容
            String content = loadFieldValue(leafReader, docId, field);
            if (content != null && content.length() > 0) {
                // 执行高亮处理
                Object highlightResult = fieldHighlighter.highlightFieldForDoc(leafReader, docId, content);
                // 存储结果...
            }
        }
    }
    
    return resultMap;
}
```

### 2. FieldHighlighter.highlightOffsetsEnums() 片段生成核心

```java
// 位置：org.apache.lucene.search.uhighlight.FieldHighlighter
protected Passage[] highlightOffsetsEnums(OffsetsEnum offsetsEnums) throws IOException {
    
    final int contentLength = breakIterator.getText().getEndIndex();
    if (offsetsEnums.length() == 0) {
        return new Passage[0];
    }
    
    // 1. 初始化优先队列，维护最佳片段
    PriorityQueue<Passage> passageQueue = new PriorityQueue<>(
        maxPassages + 1, 
        (a, b) -> Float.compare(a.getScore(), b.getScore())
    );
    
    Passage passage = new Passage(); // 当前正在构建的片段
    int pos = breakIterator.first();
    int lastPos = pos;
    
    // 2. 遍历所有偏移量，构建片段
    while (offsetsEnums.nextPosition()) {
        int start = offsetsEnums.startOffset();
        int end = offsetsEnums.endOffset();
        
        // 移动到包含当前偏移量的片段边界
        while (pos <= start && pos != BreakIterator.DONE) {
            lastPos = pos;
            pos = breakIterator.next();
        }
        
        // 检查是否需要开始新片段
        if (passage.getStartOffset() == -1) {
            // 开始新片段
            passage.setStartOffset(lastPos);
            passage.setEndOffset(pos);
        } else if (start >= passage.getEndOffset()) {
            // 当前匹配超出片段范围，需要评分并可能保存当前片段
            if (passage.getNumMatches() > 0) {
                maybeAddPassage(passageQueue, passageScorer, passage, contentLength);
            }
            // 重置片段
            passage.reset();
            passage.setStartOffset(lastPos);
            passage.setEndOffset(pos);
        }
        
        // 3. 添加匹配到当前片段
        BytesRef term = offsetsEnums.getTerm();
        int termFreqInDoc = offsetsEnums.getTermFreqInDoc();
        passage.addMatch(start, end, term, termFreqInDoc);
    }
    
    // 4. 处理最后一个片段
    if (passage.getNumMatches() > 0) {
        maybeAddPassage(passageQueue, passageScorer, passage, contentLength);
    }
    
    // 5. 从优先队列中提取最终结果
    Passage[] passages = passageQueue.toArray(new Passage[passageQueue.size()]);
    
    // 6. 按文档位置排序（用于最终输出）
    Arrays.sort(passages, Comparator.comparing(Passage::getStartOffset));
    
    return passages;
}

private void maybeAddPassage(PriorityQueue<Passage> passageQueue, PassageScorer scorer, 
                           Passage passage, int contentLength) {
    // 计算片段评分
    passage.setScore(scorer.score(passage, contentLength));
    
    // 维护固定大小的优先队列
    if (passageQueue.size() < maxPassages) {
        passageQueue.add(passage.clone());
    } else if (passage.getScore() > passageQueue.peek().getScore()) {
        passageQueue.poll();
        passageQueue.add(passage.clone());
    }
}
```

### 3. DefaultPassageFormatter.format() 格式化实现

```java
// 位置：org.apache.lucene.search.uhighlight.DefaultPassageFormatter
@Override
public String format(Passage[] passages, String content) {
    StringBuilder sb = new StringBuilder();
    int pos = 0;
    
    for (Passage passage : passages) {
        // 1. 添加省略号（如果片段不连续）
        if (!sb.isEmpty() && passage.getStartOffset() != pos) {
            sb.append(ellipsis);
        }
        
        pos = passage.getStartOffset();
        
        // 2. 处理片段内的所有匹配
        for (int i = 0; i < passage.getNumMatches(); i++) {
            int start = passage.getMatchStarts()[i];
            int end = passage.getMatchEnds()[i];
            
            // 添加匹配前的普通文本
            append(sb, content, pos, start);
            
            // 处理重叠的匹配项
            while (i + 1 < passage.getNumMatches() && 
                   passage.getMatchStarts()[i + 1] < end) {
                end = Math.max(end, passage.getMatchEnds()[++i]);
            }
            end = Math.min(end, passage.getEndOffset());
            
            // 3. 添加高亮标记
            sb.append(preTag);
            append(sb, content, start, end);
            sb.append(postTag);
            
            pos = end;
        }
        
        // 4. 添加片段结尾的普通文本
        append(sb, content, pos, Math.max(pos, passage.getEndOffset()));
        pos = passage.getEndOffset();
    }
    
    return sb.toString();
}

protected void append(StringBuilder dest, String content, int start, int end) {
    if (escape) {
        // HTML转义处理
        for (int i = start; i < end; i++) {
            char ch = content.charAt(i);
            switch (ch) {
                case '&': dest.append("&amp;"); break;
                case '<': dest.append("&lt;"); break;
                case '>': dest.append("&gt;"); break;
                case '"': dest.append("&quot;"); break;
                case '\'': dest.append("&#x27;"); break;
                case '/': dest.append("&#x2F;"); break;
                default: dest.append(ch);
            }
        }
    } else {
        dest.append(content, start, end);
    }
}
```

## 性能优化实践

### 1. 索引时优化

```java
// 为高亮优化的索引配置
public void optimizedIndexing() throws IOException {
    IndexWriterConfig config = new IndexWriterConfig(new StandardAnalyzer());
    IndexWriter writer = new IndexWriter(directory, config);
    
    // 配置字段类型以支持高效高亮
    FieldType textFieldType = new FieldType();
    textFieldType.setIndexOptions(IndexOptions.DOCS_AND_FREQS_AND_POSITIONS_AND_OFFSETS);
    textFieldType.setStoreTermVectors(true);
    textFieldType.setStoreTermVectorPositions(true);
    textFieldType.setStoreTermVectorOffsets(true);
    textFieldType.setStored(true);
    
    Document doc = new Document();
    doc.add(new Field("content", "Your document content here", textFieldType));
    doc.add(new StoredField("title", "Document Title"));
    
    writer.addDocument(doc);
    writer.close();
}
```

### 2. 查询时性能优化

```java
public class PerformanceOptimizedHighlighter {
    
    private final UnifiedHighlighter highlighter;
    private final LoadingCache<String, String[]> highlightCache;
    
    public PerformanceOptimizedHighlighter(IndexSearcher searcher, Analyzer analyzer) {
        // 1. 配置高性能的高亮器
        this.highlighter = new UnifiedHighlighter(searcher, analyzer) {
            @Override
            protected FieldOffsetStrategy getFieldOffsetStrategy(String field, Query query, Set<HighlightFlag> flags) {
                // 优先使用PostingsOffsetStrategy以获得最佳性能
                if (flags.contains(HighlightFlag.OFFSET_STRATEGY_POSTINGS)) {
                    return new PostingsOffsetStrategy(field, query, this);
                }
                // 其他策略...
                return super.getFieldOffsetStrategy(field, query, flags);
            }
        };
        
        // 2. 配置结果缓存
        this.highlightCache = CacheBuilder.newBuilder()
            .maximumSize(1000)
            .expireAfterWrite(10, TimeUnit.MINUTES)
            .build();
    }
    
    public String[] highlightWithCache(String field, Query query, TopDocs topDocs) {
        String cacheKey = generateCacheKey(field, query, topDocs);
        
        return highlightCache.get(cacheKey, () -> {
            return highlighter.highlight(field, query, topDocs);
        });
    }
    
    // 批量高亮以减少重复计算
    public Map<String, String[]> batchHighlight(String[] fields, Query query, TopDocs topDocs) {
        // 一次性处理多个字段，避免重复的文档加载
        return highlighter.highlightFields(fields, query, topDocs);
    }
}
```

### 3. 内存使用优化

```java
public class MemoryOptimizedHighlighter {
    
    public void efficientHighlighting() throws IOException {
        UnifiedHighlighter highlighter = new UnifiedHighlighter(searcher, analyzer) {
            
            @Override
            protected FieldOffsetStrategy getFieldOffsetStrategy(String field, Query query, Set<HighlightFlag> flags) {
                // 对于大文档，使用AnalysisOffsetStrategy减少内存占用
                if (isLargeDocument(field)) {
                    return new AnalysisOffsetStrategy(field, query, this);
                }
                return super.getFieldOffsetStrategy(field, query, flags);
            }
            
            @Override
            protected PassageFormatter getFormatter(String field) {
                // 使用对象池减少GC压力
                return new PooledPassageFormatter();
            }
        };
        
        // 限制最大分析长度以控制内存使用
        highlighter.setMaxLength(100000);
        
        // 使用流式处理大量文档
        processDocumentsInBatches(highlighter, query);
    }
    
    private void processDocumentsInBatches(UnifiedHighlighter highlighter, Query query) {
        final int batchSize = 50;
        // 分批处理文档，避免一次性加载过多内容到内存
        // 实现细节...
    }
}
```

这些代码示例展示了 Lucene Highlighter 的实际应用方法，从基础使用到高级配置，再到性能优化策略，为开发者提供了完整的实践指南。