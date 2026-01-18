# CLAUDE.md

This file provides guidance to Claude Code when working with the AegisLab documentation repository.

## Project Overview

This is the documentation repository for the AegisLab ecosystem, built with Hugo and the Hextra theme. The documentation follows a persona-based structure with two main user paths:

1. **Algorithm Developers**: Users who develop and evaluate RCA algorithms
2. **Dataset Creators**: Users who create datasets through fault injection

## Hugo/Hextra Structure Rules

### Critical: Directory Structure Pattern

**IMPORTANT**: This documentation uses Hugo's content organization pattern. Each page MUST be represented as a directory containing only `_index.md`, not as standalone `.md` files.

**✅ Correct Structure:**
```
content/
├── algorithm-developers/
│   ├── _index.md
│   ├── quickstart/
│   │   └── _index.md
│   ├── local-evaluation/
│   │   └── _index.md
│   └── remote-evaluation/
│       ├── _index.md
│       ├── upload-algorithm/
│       │   └── _index.md
│       └── submit-execution/
│           └── _index.md
```

**❌ Incorrect Structure:**
```
content/
├── algorithm-developers/
│   ├── _index.md
│   ├── quickstart.md          # WRONG: standalone .md file
│   ├── local-evaluation.md    # WRONG: standalone .md file
│   └── remote-evaluation/
│       ├── _index.md
│       ├── upload-algorithm.md    # WRONG: standalone .md file
│       └── submit-execution.md    # WRONG: standalone .md file
```

### Structure Validation

Before making changes, verify the structure:

```bash
# Check for any standalone .md files (should return 0)
find content/algorithm-developers content/dataset-creators -type f -name "*.md" ! -name "_index.md" | wc -l
```

### Converting Standalone Files

If you need to create a new page, always create it as a directory with `_index.md`:

```bash
# Create new page
mkdir -p content/section-name/page-name
# Create content in _index.md
cat > content/section-name/page-name/_index.md << 'EOF'
---
title: Page Title
weight: 1
---

# Page Content
EOF
```

## Documentation Structure

### Persona-Based Organization

```
content/
├── _index.md                    # Landing page with persona selection
├── algorithm-developers/        # Algorithm developer path
│   ├── quickstart/
│   ├── development-guide/
│   ├── local-evaluation/
│   ├── remote-evaluation/
│   ├── examples/
│   ├── reference/
│   └── troubleshooting/
├── dataset-creators/            # Dataset creator path
│   ├── quickstart/
│   ├── using-aegislab/
│   ├── fault-injection-guide/
│   ├── monitoring-collection/
│   ├── dataset-management/
│   ├── advanced-topics/
│   └── reference/
├── concepts/                    # Shared concepts
├── deployment/                  # Shared deployment guides
├── getting-started/             # Shared getting started
└── contributing/                # Shared contributing guide
```

### Frontmatter Requirements

Every `_index.md` file must include proper frontmatter:

```yaml
---
title: Page Title
weight: 1  # Controls ordering in navigation
---
```

## Content Guidelines

### Code Examples

- Always include working code examples
- Provide context and explanations
- Show expected output
- Include error handling examples

### Cross-References

Use relative links to reference other pages:

```markdown
See [Remote Evaluation](../remote-evaluation) for details.
```

### Troubleshooting Sections

Most guides should include a troubleshooting section with:
- Common errors
- Solutions
- Verification steps

## Documentation Review with doc-coauthoring

### When to Use doc-coauthoring

Use the `/doc-coauthoring` skill to review and improve documentation:

```bash
# Review the entire documentation repository
/doc-coauthoring
```

The doc-coauthoring skill helps with:
- **Structure review**: Verify organization and navigation
- **Content quality**: Check clarity, completeness, and accuracy
- **Consistency**: Ensure consistent terminology and formatting
- **User experience**: Validate that documentation serves user needs
- **Gap analysis**: Identify missing content or unclear sections

### Documentation Review Workflow

1. **Initial Review**: Use doc-coauthoring to assess current state
2. **Identify Issues**: Note structural problems, content gaps, unclear sections
3. **Prioritize Changes**: Focus on high-impact improvements
4. **Iterate**: Make changes and re-review
5. **Validate**: Ensure changes maintain Hugo/Hextra structure

### Review Checklist

When reviewing documentation:

- [ ] All pages follow `directory/_index.md` structure
- [ ] Frontmatter is present and correct
- [ ] Code examples are working and complete
- [ ] Cross-references are valid
- [ ] Navigation is logical and clear
- [ ] Both persona paths are self-contained
- [ ] Troubleshooting sections are comprehensive
- [ ] Terminology is consistent

## Common Tasks

### Adding a New Page

```bash
# 1. Create directory
mkdir -p content/section-name/new-page

# 2. Create _index.md
cat > content/section-name/new-page/_index.md << 'EOF'
---
title: New Page Title
weight: 5
---

# New Page Title

Content here...
EOF

# 3. Verify structure
find content -name "*.md" ! -name "_index.md"  # Should return nothing
```

### Updating Existing Content

1. Read the existing `_index.md` file
2. Make changes using Edit tool
3. Verify frontmatter is preserved
4. Test navigation locally with `hugo server`

### Reorganizing Content

When moving pages:
1. Maintain the `directory/_index.md` structure
2. Update all cross-references
3. Update parent `_index.md` files
4. Verify navigation still works

## Testing Documentation

### Local Preview

```bash
# Start Hugo server
hugo server

# Navigate to http://localhost:1313
```

### Structure Validation

```bash
# Verify no standalone .md files
find content/algorithm-developers content/dataset-creators -type f -name "*.md" ! -name "_index.md"

# Should output nothing if structure is correct
```

### Link Validation

```bash
# Check for broken links (if hugo-link-checker is installed)
hugo && htmltest
```

## Best Practices

1. **Always use directory/_index.md structure**: Never create standalone `.md` files
2. **Test locally**: Preview changes with `hugo server` before committing
3. **Maintain consistency**: Follow existing patterns for frontmatter and formatting
4. **Use doc-coauthoring**: Regularly review documentation quality
5. **Keep personas separate**: Minimize cross-referencing between algorithm-developers and dataset-creators paths
6. **Include examples**: Every guide should have working code examples
7. **Add troubleshooting**: Help users solve common problems
8. **Update cross-references**: When moving or renaming pages, update all links

## Troubleshooting

### Hugo Build Fails

Check for:
- Missing frontmatter in `_index.md` files
- Invalid YAML in frontmatter
- Broken cross-references

### Navigation Not Working

Verify:
- All pages have `weight` in frontmatter
- Directory structure is correct
- Parent `_index.md` files exist

### Content Not Rendering

Check:
- File is named `_index.md` (not `index.md` or other)
- Frontmatter is properly formatted
- No standalone `.md` files in the directory

## Related Documentation

- [Hugo Documentation](https://gohugo.io/documentation/)
- [Hextra Theme](https://imfing.github.io/hextra/)
- Main project CLAUDE.md: `/home/nn/workspace/proj/CLAUDE.md`

## Maintenance

When making significant structural changes:
1. Use doc-coauthoring skill to review changes
2. Verify all pages follow directory/_index.md pattern
3. Test navigation locally
4. Update this CLAUDE.md if new patterns emerge
