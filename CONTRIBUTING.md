# Contributing to DroidCast

Thank you for your interest in contributing to DroidCast! 🎉

## How to Contribute

### Reporting Bugs
1. Check existing [Issues](https://github.com/BambangSB/DroidCast/issues) first
2. Open a new issue with:
   - Device model and Android version
   - PC OS version
   - Steps to reproduce
   - Expected vs actual behavior
   - Logs/screenshots if available

### Feature Requests
Open an issue with the `enhancement` label describing:
- What feature you'd like
- Why it would be useful
- Any implementation ideas

### Pull Requests
1. Fork the repository
2. Create a feature branch (`git checkout -b feature/amazing-feature`)
3. Commit your changes (`git commit -m 'Add amazing feature'`)
4. Push to the branch (`git push origin feature/amazing-feature`)
5. Open a Pull Request

## Development Setup

### Android App (Flutter)
```bash
cd android
flutter pub get
flutter run
```

### PC Companion (Python)
```bash
cd pc-companion
pip install -r requirements.txt
python main.py
```

## Code Style
- **Dart**: Follow [Effective Dart](https://dart.dev/effective-dart) guidelines
- **Python**: Follow [PEP 8](https://pep8.org/) style guide
- **Kotlin**: Follow [Kotlin Coding Conventions](https://kotlinlang.org/docs/coding-conventions.html)
