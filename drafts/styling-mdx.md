/* mdx-content.css */

/* Headings */
.mdx-content h1 {
  font-size: 2.25rem;
  font-weight: 700;
  line-height: 2.5rem;
  margin-top: 0;
  margin-bottom: 2rem;
}

.mdx-content h2 {
  font-size: 1.875rem;
  font-weight: 600;
  margin-top: 3rem;
  margin-bottom: 1rem;
}

.mdx-content h3 {
  font-size: 1.5rem;
  font-weight: 600;
  margin-top: 2rem;
  margin-bottom: 0.75rem;
}

/* Paragraphs */
.mdx-content p {
  margin-bottom: 1.25rem;
  line-height: 1.75;
}

/* Lists */
.mdx-content ul,
.mdx-content ol {
  margin-bottom: 1.25rem;
  padding-left: 1.5rem;
}

.mdx-content li {
  margin-bottom: 0.5rem;
}

/* Links */
.mdx-content a {
  color: #2563eb;
  text-decoration: underline;
}

.mdx-content a:hover {
  color: #1d4ed8;
}

/* Inline code */
.mdx-content code {
  background-color: #f3f4f6;
  padding: 0.125rem 0.375rem;
  border-radius: 0.25rem;
  font-size: 0.875em;
  font-family: monospace;
}

/* Code blocks */
.mdx-content pre {
  background-color: #1f2937;
  color: #f9fafb;
  padding: 1rem;
  border-radius: 0.5rem;
  overflow-x: auto;
  margin-bottom: 1.25rem;
}

.mdx-content pre code {
  background-color: transparent;
  padding: 0;
  color: inherit;
}

/* Blockquotes */
.mdx-content blockquote {
  border-left: 4px solid #e5e7eb;
  padding-left: 1rem;
  margin-left: 0;
  margin-bottom: 1.25rem;
  font-style: italic;
  color: #6b7280;
}

/* Tables */
.mdx-content table {
  width: 100%;
  border-collapse: collapse;
  margin-bottom: 1.25rem;
}

.mdx-content th,
.mdx-content td {
  border: 1px solid #e5e7eb;
  padding: 0.75rem;
  text-align: left;
}

.mdx-content th {
  background-color: #f9fafb;
  font-weight: 600;
}

/* Horizontal rule */
.mdx-content hr {
  border: none;
  border-top: 1px solid #e5e7eb;
  margin: 2rem 0;
}

/* Images */
.mdx-content img {
  max-width: 100%;
  height: auto;
  border-radius: 0.5rem;
  margin-bottom: 1.25rem;
}

<div className="mdx-content">
  {/* Your MDX renders here */}
</div>