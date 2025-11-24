## Get Started with MkDocs + Material

### 1. **Install MkDocs & Material**

   ```bash
   pip install mkdocs-material
   ```

### 2. **Create a docs folder**

   ```
   mkdocs new my-docs
   cd my-docs
   ```

### 3. **Configure `mkdocs.yml`** (example):

   ```yaml
   site_name: "My Developer Guide"
   theme:
     name: material

   nav:
     - Home: index.md
     - Schema Design: schema-design.md
     - Edge Cases: edge-cases.md
     - API Reference: api.md

   markdown_extensions:
     - codehilite
     - pymdownx.superfences
   ```

### 4. **Write your Markdown** files with code snippets, bullet points, tables, etc.

### 5. **Preview locally**:

   ```bash
   mkdocs serve
   ```

### 6. **Deploy**: Use GitHub Pages:

   * Create GitHub repo
   * Add `mkdocs.yml` + `docs/`
   * Use GitHub Actions or `mkdocs gh-deploy` to publish

---
