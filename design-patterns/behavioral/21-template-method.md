# Pattern 21 — Template Method
**Category:** Behavioral

---

## Intent

Define the **skeleton of an algorithm** in a base class, deferring some steps to subclasses. Template Method lets subclasses redefine certain steps of an algorithm without changing the algorithm's structure.

---

## The Problem It Solves

Two data mining programs extract data from different sources (CSV and XML), but the overall process is the same: open → parse → extract → analyse → report. The structure shouldn't be duplicated; only the specific steps differ.

---

## Java Implementation

```java
// Abstract class with template method
public abstract class DataProcessor {

    // TEMPLATE METHOD — defines the algorithm skeleton, final so it can't be overridden
    public final ProcessingResult process(String source) {
        System.out.println("=== Starting data processing ===");
        RawData    raw       = openDataSource(source);      // abstract
        ParsedData parsed    = parseData(raw);               // abstract
        Data       extracted = extractData(parsed);          // abstract
        Data       filtered  = filterData(extracted);        // hook (optional override)
        Report     report    = generateReport(filtered);     // concrete (shared)
        closeDataSource(raw);                                // concrete (shared)
        return new ProcessingResult(report);
    }

    // Abstract steps — subclasses MUST implement
    protected abstract RawData    openDataSource(String source);
    protected abstract ParsedData parseData(RawData raw);
    protected abstract Data       extractData(ParsedData parsed);

    // Hook methods — subclasses MAY override (have default implementation)
    protected Data filterData(Data data) {
        System.out.println("Applying default filter (no-op)");
        return data;
    }

    // Concrete steps — shared by all subclasses
    private Report generateReport(Data data) {
        System.out.println("Generating standardised report");
        return new Report(data);
    }

    private void closeDataSource(RawData raw) {
        System.out.println("Closing data source");
    }
}

// Concrete implementations
public class CsvDataProcessor extends DataProcessor {
    @Override
    protected RawData openDataSource(String path) {
        System.out.println("Opening CSV file: " + path);
        return new RawData(path);
    }
    @Override
    protected ParsedData parseData(RawData raw) {
        System.out.println("Parsing CSV with comma delimiter");
        return new ParsedData();
    }
    @Override
    protected Data extractData(ParsedData parsed) {
        System.out.println("Extracting columns from CSV rows");
        return new Data();
    }
}

public class XmlDataProcessor extends DataProcessor {
    @Override
    protected RawData openDataSource(String path) {
        System.out.println("Opening XML file: " + path);
        return new RawData(path);
    }
    @Override
    protected ParsedData parseData(RawData raw) {
        System.out.println("Parsing XML DOM tree");
        return new ParsedData();
    }
    @Override
    protected Data extractData(ParsedData parsed) {
        System.out.println("Extracting elements using XPath");
        return new Data();
    }
    @Override
    protected Data filterData(Data data) {
        System.out.println("XML: Removing namespace prefixes");
        return data;
    }
}

// Usage
DataProcessor csvProcessor = new CsvDataProcessor();
csvProcessor.process("report.csv");

DataProcessor xmlProcessor = new XmlDataProcessor();
xmlProcessor.process("data.xml");
```

---

## Real-World Use Cases

- **Spring Batch** — `AbstractItemCountingItemStreamItemReader` defines the template; subclasses implement `doRead()`
- **Spring's JdbcTemplate** — template method controls connection lifecycle; callback provides SQL
- **Java's `AbstractList`** — `get()` and `size()` are abstract; everything else (iterator, subList, etc.) is concrete
- **HttpServlet** — `service()` is the template; `doGet()`, `doPost()` are the steps

## When to Use

- When you want to let subclasses implement varying behaviour while keeping shared structure in one place
- When common behaviour across multiple subclasses should be factored into a shared class to avoid duplication
- When you want to control which steps subclasses may extend (hooks vs abstract methods)

---

| | |
|---|---|
| [← Strategy](./20-strategy.md) | [Next → Visitor](./22-visitor.md) |

---
---

