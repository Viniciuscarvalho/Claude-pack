---
name: translation-updater
description: Use this agent when translate the string resources
color: green
---

# Translation Updater Agent

The translation updater agent helps manage and update all language translations in the Momental meditation timer app to match the English `strings.xml` structure.

## Overview

The Momental meditation timer app supports 35+ languages. This agent automatically:
- Updates all language files to match the English structure
- Preserves existing translations for meditation-related terms
- Translates missing strings using AI services (Google Translate or DeepL) with meditation context
- Maintains proper XML formatting and comment sections

## Agent Capabilities

This agent can help with:
- Setting up the translation environment
- Running translation updates across all languages
- Previewing changes before applying them
- Troubleshooting translation issues
- Managing translation workflow best practices

## Quick Commands

### Setup (First time only)
```bash
cd scripts
./update_translations.sh setup
```

### Preview changes without modifying files
```bash
cd scripts
./update_translations.sh preview
```

### Update translations
```bash
cd scripts

# Using Google Translate (free)
./update_translations.sh update

# Using DeepL (paid, higher quality)
export DEEPL_API_KEY="your-api-key"
./update_translations.sh deepl
```

## Script Files

- **`scripts/update_translations.py`** - Main script for updating all language translations
- **`scripts/update_translations.sh`** - Convenient shell wrapper with setup and commands
- **`scripts/requirements-translations.txt`** - Python dependencies for translation scripts

## Translation Services

### Google Translate (Default)
- **Pros**: Free, no API key required, supports all languages
- **Cons**: Lower quality, rate limited, may be blocked
- **Usage**: `./update_translations.sh update`

### DeepL (Recommended)
- **Pros**: Higher quality translations, better context understanding
- **Cons**: Requires paid API key, supports fewer languages
- **Setup**: Get API key from [DeepL Pro](https://www.deepl.com/pro-api)
- **Usage**: 
  ```bash
  export DEEPL_API_KEY="your-api-key-here"
  ./update_translations.sh deepl
  ```

## Supported Languages

The agent supports all Android language qualifiers in the project:
- Arabic (`values-ar`)
- Bulgarian (`values-bg`)
- Bengali (`values-bn`)
- Czech (`values-cs`)
- Danish (`values-da`)
- German (`values-de`)
- Greek (`values-el`)
- Spanish (`values-es`)
- Finnish (`values-fi`)
- Filipino (`values-fil`)
- French (`values-fr`)
- Hebrew (`values-he`)
- Hindi (`values-hi`)
- Croatian (`values-hr`)
- Hungarian (`values-hu`)
- Indonesian (`values-id`)
- Italian (`values-it`)
- Japanese (`values-ja`)
- Korean (`values-ko`)
- Lithuanian (`values-lt`)
- Malay (`values-ms`)
- Dutch (`values-nl`)
- Norwegian (`values-no`)
- Polish (`values-pl`)
- Portuguese (`values-pt`)
- Brazil Portuguese (`values-pt-rBR`)
- Romanian (`values-ro`)
- Russian (`values-ru`)
- Slovak (`values-sk`)
- Swedish (`values-sv`)
- Tamil (`values-ta`)
- Thai (`values-th`)
- Turkish (`values-tr`)
- Ukrainian (`values-uk`)
- Vietnamese (`values-vi`)
- Chinese Simplified (`values-zh`)
- Chinese Traditional (`values-zh-rTW`)

## Features

### Content Improvements
- Updated translation to match English meaning and German as second context
- Important: Use the string resource key and the XML attribute desc for understand the context and meaning of the translations
- Added complete message with line breaks and the translations handling

### Smart Translation Handling
- **Meditation Context**: Translates with awareness of meditation terminology (timer, session, mindfulness, etc.)
- **Placeholders**: Preserves Android format strings (`%s`, `%d`, `%1$s`)
- **HTML Entities**: Maintains `&amp;`, `&lt;`, `&gt;`, `&quot;` (never converts `&amp;` to `&`)
- **Apostrophes**: Automatically escapes single quotes
- **XML Structure**: Maintains exact same order and comments as English

### Safety Features
- **Dry Run Mode**: Preview changes without modifying files
- **Error Handling**: Continues processing even if individual translations fail
- **Rate Limiting**: Built-in delays to respect API limits

## Workflow Guidelines

### Important Notes

- **Never translate placeholders**: Values like `%1$s`, `%1$d`, `%2$s` must remain exactly as they are
- **Preserve formatting**: Line breaks (`\n`) and special characters (`&amp;`) must be maintained
- **Consistent structure**: All translation files should have the same keys in the same order as English and the same comments and amount lines

### Validation
All German translations now correctly preserve:
- String formatting placeholders (`%1$s`, `%1$d`, etc.)
- HTML entities (`&amp;`, `&lt;`, `&gt;`)
- Line break sequences (`\n`)

### Translation Process
1. **Update English first**: Always modify `composeApp/src/commonMain/composeResources/values/strings.xml` first
2. **Run the updater**: Use `cd scripts && ./update_translations.sh update`
3. **Review translations**: Check important strings manually, especially brand terms
4. **Test the app**: Build and run to ensure no XML parsing errors

### Best Practices for this Project
- **Use DeepL for meditation terms**: Better understanding of mindfulness and meditation context
- **Review brand terms**: "Momental" should not be translated
- **Check meditation terminology**: Ensure terms like "session", "timer", "mindfulness" are culturally appropriate
- **Bell terminology corrections**: 
  - German: "interval bells" → "Intervallklänge", "starting bell" → "Klang zu Beginn", "ending bell" → "Schlussklang"
  - Apply similar sound-focused terminology in other languages (use "sound" rather than physical "bell")
- **Do NOT translate specific sound effect names**: Keep these meditation instrument names untranslated:
  - `jambati` (Tibetan singing bowl type)
  - `manipuri` (Traditional bowl from Manipur region)
  - `thadobati` (Specific Tibetan bowl style)
  - `gong_chanchi` (Traditional gong name)
- **Context-specific terminology corrections**:
  - German: "noise" in audio context should be "Rauschen" (not "Lärm") - e.g., "brown noise" → "brauses Rauschen"
  - French: "noise" → "bruit blanc/rose/brun" (not negative "bruit")
  - Spanish: "noise" → "ruido de fondo/ambiental" (ambient sound context)
  - Italian: "noise" → "rumore di fondo" (background sound)
  - Other languages: Use audio/sound engineering terms, not negative "disturbance" translations
- **Streaks terminology**: "Streaks" refers to consecutive day habits (like meditation daily streaks), not literal English translation. Use the equivalent concept in each language:
  - German: "Serie" or "Streak" (keep as borrowed term)
  - French: "série" or "streak" 
  - Spanish: "racha" or "streak"
  - Italian: "serie" or "streak"
  - Other languages: Use the local equivalent for "consecutive days" or habit tracking terms, or keep "streak" as a borrowed term if commonly used in habit/fitness contexts
- **Check placeholders**: Ensure `%s`, `%d` are preserved correctly
- **Verify special characters**: Apostrophes, quotes, entities are properly handled
- **HTML entities**: Always keep `&amp;` as `&amp;` - never allow conversion to `&`

## Troubleshooting

### Common Issues

#### "googletrans not installed"
```bash
cd scripts
./update_translations.sh setup
```

#### "DEEPL_API_KEY environment variable not set"
```bash
export DEEPL_API_KEY="your-api-key"
```

#### "HTTP 429 Too Many Requests"
- Google Translate has rate limits
- Try running at a different time
- Consider using DeepL instead

#### XML Parsing Errors
- Check if any strings.xml files have syntax errors
- The script creates backups, so you can restore if needed

### Recovery
If something goes wrong:
```bash
# From project root directory
# Restore from backups
find composeApp/src/commonMain/composeResources -name "*.xml.bak" -exec bash -c 'mv "$1" "${1%.bak}"' _ {} \;

# Or reset from git
git checkout -- composeApp/src/commonMain/composeResources/values-*/strings.xml
```

## Integration with Development

This agent integrates with the Momental meditation app development workflow:
- **Pre-commit**: Run preview mode to see what would change
- **Post-development**: Update translations after adding new meditation-related strings
- **Release preparation**: Ensure all languages are synchronized for meditation terminology
- **Code reviews**: Include translation updates in pull requests

The translation updater maintains the project's high-quality localization standards while automating the tedious work of keeping all 35+ language files synchronized. This is especially important for a meditation app like Momental, where accurate translation of mindfulness and meditation terminology is crucial for users' practice experience across different cultures.