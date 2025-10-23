# Translation Status

This document tracks which pages have Vietnamese translations.

## ✅ Translated Pages (Vietnamese Available)

| English File | Vietnamese File | Status |
|--------------|-----------------|--------|
| `README.md` | `README.vi.md` | ✅ Complete |
| `00-overview.md` | `00-overview.vi.md` | ✅ Complete |
| `SETUP.md` | `SETUP.vi.md` | ✅ Complete |

## 🔄 Not Yet Translated (Falls back to English)

| English File | Vietnamese File Needed | Priority |
|--------------|------------------------|----------|
| `01-instructions.md` | `01-instructions.vi.md` | High |
| `02-state-accounts.md` | `02-state-accounts.vi.md` | Medium |
| `03-bonding-curve-math.md` | `03-bonding-curve-math.vi.md` | Medium |
| `04-fee-system.md` | `04-fee-system.vi.md` | High |
| `05-workflows.md` | `05-workflows.vi.md` | High |
| `06-migration.md` | `06-migration.vi.md` | Medium |

## 📝 How to Add Translations

1. Copy an English file:
   ```bash
   cp docs/01-instructions.md docs/01-instructions.vi.md
   ```

2. Translate the content in the `.vi.md` file

3. The i18n plugin will automatically:
   - Detect the `.vi.md` file
   - Show Vietnamese content when language is switched to Vietnamese
   - Fall back to English if no `.vi.md` version exists

## 🌐 Translation Guidelines

### What to Translate
- ✅ Headings and titles
- ✅ Body text and descriptions
- ✅ Table contents
- ✅ Comments in code examples (optional)
- ✅ Admonition text (notes, warnings, etc.)

### What NOT to Translate
- ❌ Code blocks (keep as-is)
- ❌ Variable names
- ❌ File paths
- ❌ URLs
- ❌ Program IDs
- ❌ Technical constants

### Example

**English (01-instructions.md):**
```markdown
## Admin Instructions

### create_claim_fee_operator

Creates an operator account.

**Parameters**: None
```

**Vietnamese (01-instructions.vi.md):**
```markdown
## Lệnh Admin

### create_claim_fee_operator

Tạo một tài khoản operator.

**Tham số**: Không có
```

## 📊 Translation Progress

- Total pages: 9
- Translated: 3 (33%)
- Remaining: 6 (67%)

## 🎯 Next Priority

Focus on translating these high-priority pages first:
1. `01-instructions.vi.md` - Most frequently accessed
2. `05-workflows.vi.md` - Practical examples
3. `04-fee-system.vi.md` - Important for users

## 🤝 Contributing

To contribute translations:
1. Fork the repository
2. Create `.vi.md` versions of pages
3. Submit a pull request
4. We'll review and merge!

Thank you for helping make this documentation accessible to Vietnamese speakers! 🇻🇳
