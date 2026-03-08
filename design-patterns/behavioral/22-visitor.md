# Pattern 22 — Visitor
**Category:** Behavioral

---

## Intent

Represent an operation to be performed on elements of an object structure. Visitor lets you define a new operation **without changing the classes** of the elements on which it operates.

---

## The Problem It Solves

You have a document model (Paragraph, Image, Table, Heading). You want to add operations like export to HTML, export to PDF, spell-check, and word-count — without modifying the document element classes each time.

---

## Java Implementation

```java
// Visitor interface — one visit method per element type
public interface DocumentVisitor {
    void visitParagraph(Paragraph paragraph);
    void visitImage(Image image);
    void visitTable(Table table);
    void visitHeading(Heading heading);
}

// Element interface
public interface DocumentElement {
    void accept(DocumentVisitor visitor);  // "double dispatch"
}

// Concrete Elements
public class Paragraph implements DocumentElement {
    private final String text;
    public Paragraph(String text) { this.text = text; }
    public String getText() { return text; }
    @Override
    public void accept(DocumentVisitor visitor) { visitor.visitParagraph(this); }
}

public class Image implements DocumentElement {
    private final String url;
    private final int width, height;
    public Image(String url, int w, int h) { this.url = url; width = w; height = h; }
    public String getUrl() { return url; }
    public int getWidth()  { return width; }
    public int getHeight() { return height; }
    @Override
    public void accept(DocumentVisitor visitor) { visitor.visitImage(this); }
}

public class Heading implements DocumentElement {
    private final String text;
    private final int level;
    public Heading(String text, int level) { this.text = text; this.level = level; }
    public String getText()  { return text; }
    public int    getLevel() { return level; }
    @Override
    public void accept(DocumentVisitor visitor) { visitor.visitHeading(this); }
}

// Concrete Visitors — new operations without touching elements
public class HtmlExportVisitor implements DocumentVisitor {
    private final StringBuilder html = new StringBuilder();

    @Override
    public void visitParagraph(Paragraph p) {
        html.append("<p>").append(p.getText()).append("</p>\n");
    }
    @Override
    public void visitImage(Image img) {
        html.append(String.format("<img src=\"%s\" width=\"%d\" height=\"%d\"/>%n",
                img.getUrl(), img.getWidth(), img.getHeight()));
    }
    @Override
    public void visitTable(Table t) { html.append("<table>...</table>\n"); }
    @Override
    public void visitHeading(Heading h) {
        html.append("<h").append(h.getLevel()).append(">")
            .append(h.getText())
            .append("</h").append(h.getLevel()).append(">\n");
    }
    public String getHtml() { return html.toString(); }
}

public class WordCountVisitor implements DocumentVisitor {
    private int count = 0;

    @Override
    public void visitParagraph(Paragraph p) { count += p.getText().split("\\s+").length; }
    @Override
    public void visitHeading(Heading h)     { count += h.getText().split("\\s+").length; }
    @Override
    public void visitImage(Image img)       { /* images have no words */ }
    @Override
    public void visitTable(Table t)         { /* simplification */ }
    public int getWordCount() { return count; }
}

// Usage
List<DocumentElement> doc = List.of(
    new Heading("Introduction", 1),
    new Paragraph("This is the first paragraph."),
    new Image("photo.jpg", 800, 600)
);

HtmlExportVisitor htmlVisitor = new HtmlExportVisitor();
WordCountVisitor  wordVisitor = new WordCountVisitor();

doc.forEach(el -> el.accept(htmlVisitor));
doc.forEach(el -> el.accept(wordVisitor));

System.out.println(htmlVisitor.getHtml());
System.out.println("Words: " + wordVisitor.getWordCount());
```

---

## Real-World Use Cases

- **Java compiler AST visitors** — `javax.lang.model.element.ElementVisitor`
- **Spring's BeanDefinitionVisitor** — traverses bean definition structures
- **ANTLR parse tree visitors** — visit AST nodes for language processing
- **Hibernate's SQL AST** — visitor-based SQL generation

## When to Use

- When you need to perform many distinct and unrelated operations on an object structure
- When the object structure classes rarely change, but you often add new operations
- When you don't want to pollute element classes with unrelated operations

---

| | |
|---|---|
| [← Template Method](./21-template-method.md) | [Next → Interpreter](./23-interpreter.md) |

---
---

