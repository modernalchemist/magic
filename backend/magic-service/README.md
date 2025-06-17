# Magic Service

## Project Overview

Magic Service is a high-performance PHP microservice application based on the Hyperf framework, using the Swow coroutine driver to achieve high concurrency processing capabilities. This project integrates multiple functional modules, including AI search, chat functions, file processing, permission management, etc., aiming to provide a comprehensive service solution.

## Features

- **AI Search Function**: Integrates APIs from search engines like Google to provide intelligent search capabilities.
- **Chat System**: Supports real-time communication and session management.
- **File Processing**: File upload, download, and management functions.
- **Process Management**: Supports workflow configuration and execution.
- **Assistant Function**: Extendable assistant function support.

## Environment Requirements

- PHP >= 8.3
- Swow Extension
- Redis Extension
- PDO Extension
- Other extensions: bcmath, curl, fileinfo, openssl, xlswriter, zlib, etc.
- Composer

## Installation and Deployment

### 1. Clone Project

```bash
git clone https://github.com/dtyq/magic.git
cd magic-service
```

### 2. Install Dependencies

```bash
composer install
```




### 3. Environment Configuration

Copy the environment configuration file and modify it as needed:

```bash
cp .env.example .env
```

### Database Migration

```bash
php bin/hyperf.php migrate
```

## Running the Application

### Start Frontend Service

```bash
cd static/web && npm install && npm run dev
```

### Start Backend Service

```bash
php bin/hyperf.php start
```

Alternatively, you can use a script to start:

```bash
sh start.sh
```

## Development Guide

### Project Structure

- `app/` - Application code
  - `Application/` - Application layer code
  - `Domain/` - Domain layer code
  - `Infrastructure/` - Infrastructure layer code
  - `Interfaces/` - Interface layer code
  - `ErrorCode/` - Error code definitions
  - `Listener/` - Event listeners
- `config/` - Configuration files
- `migrations/` - Database migration files
- `test/` - Unit tests
- `bin/` - Executable scripts
- `static/` - Static resource files

### Code Standards

The project uses PHP-CS-Fixer for code style checking and fixing:

```bash
composer fix
```

Use PHPStan for static code analysis:

```bash
composer analyse
```

### Unit Tests

Use the following command to run unit tests:

```bash
vendor/bin/phpunit
# Or use
composer test
```

## Docker Deployment

The project provides a Dockerfile, and you can use the following command to build the image:

```bash
docker build -t magic-service .
```

## Contribution Guide

1. Fork Project
2. Create feature branch (`git checkout -b feature/amazing-feature`)
3. Commit changes (`git commit -m 'Add some amazing feature'`)
4. Push to branch (`git push origin feature/amazing-feature`)
5. Submit Pull Request

## License

This project is licensed under the MIT License - see the LICENSE file for details.
