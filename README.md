# MediFinderBot

A WhatsApp-based chatbot system that helps users locate available medicines at nearby medical centers in northwestern Peru. The platform connects patients to real-time inventory data from public medical facilities, making it easier to find needed medications.

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)

## 🌟 Project Overview

MediFinderBot addresses a critical healthcare access challenge in Peru by providing a simple, accessible way for patients to find where medicines are in stock. Using WhatsApp (a widely adopted messaging platform), patients can quickly locate needed medications at the nearest public medical center, saving time and reducing frustration.

**Key Features:**
- Real-time medicine availability search via WhatsApp
- Location-based queries to find the nearest available stock
- Automated data ingestion from official health system records
- RESTful API for medication inventory access

## 🏗️ System Architecture

The MediFinderBot system consists of four main components working together:

```
┌─────────────────┐      ┌─────────────────┐      ┌─────────────────┐
│                 │      │                 │      │                 │
│  Data Ingestor  │──►   │  PostgreSQL DB  │◄──── │    API Server   │
│                 │      │                 │      │                 │
└─────────────────┘      └─────────────────┘      └────────┬────────┘
                                                           │
                                                           ▼
                                                  ┌─────────────────┐
                                                  │                 │
                                                  │ WhatsApp Chatbot│
                                                  │                 │
                                                  └─────────────────┘
```

_Note: A detailed architecture diagram is available in the `docs/architecture/` directory_

## 🧩 Components

### Data Ingestor ([medifinder-ingestor](https://github.com/MediFinderBot/medifinder-ingestor))
- Scheduled ETL service that processes medicine inventory data from source files
- Handles data cleaning, normalization, and incremental updates
- Runs on a regular schedule to ensure data freshness

### API Server ([medifinder-api](https://github.com/MediFinderBot/medifinder-api))
- RESTful API service that provides secure access to medicine inventory data
- Features location-based searches and medicine availability queries
- Optimized for WhatsApp integration with low-latency responses

### WhatsApp Chatbot ([medifinder-bot](https://github.com/MediFinderBot/medifinder-bot))
- WhatsApp-based chatbot interface for user interactions
- Implements conversational workflows and natural language processing
- Manages user sessions and context for multi-turn conversations

### Database ([medifinder-db](https://github.com/MediFinderBot/medifinder-db))
- Database schema definitions, migrations, and seed data
- Includes data models, indexes, and query optimization
- PostgreSQL implementation with focus on query performance

## 🔧 Technology Stack

- **Backend:** Python 3.9+
- **Database:** PostgreSQL 13+
- **API Framework:** Flask
- **Messaging:** WhatsApp Business API
- **Deployment:** PythonAnywhere
- **CI/CD:** GitHub Actions
- **Documentation:** Markdown, Swagger/OpenAPI

## 📋 Development Roadmap

Our development is organized into 8 phases:

1. Project Setup and Infrastructure (2 weeks)
2. Data Modeling and Database Implementation (3 weeks)
3. Data Ingestion Service (2 weeks)
4. MCP Server Development (3 weeks)
5. WhatsApp Chatbot Development (4 weeks)
6. Testing and Quality Assurance (2 weeks)
7. Deployment and Monitoring (2 weeks)
8. Documentation and Handover (2 weeks)

For detailed progress tracking, see the [Project Board](https://github.com/orgs/MediFinderBot/projects).

## 🚀 Getting Started

### Prerequisites
- Python 3.9+
- PostgreSQL 13+
- PythonAnywhere account
- WhatsApp Business API access

### Local Development Setup

1. Clone the repositories:
```bash
git clone https://github.com/MediFinderBot/medifinder-api.git
git clone https://github.com/MediFinderBot/medifinder-bot.git
git clone https://github.com/MediFinderBot/medifinder-ingestor.git
git clone https://github.com/MediFinderBot/medifinder-db.git
```

2. Set up the database:
```bash
cd medifinder-db
python setup_database.py
```

3. Configure environment variables (see .env.example in each repository)

4. Run the components (each in a separate terminal):
```bash
# API Server
cd medifinder-api
python -m venv venv
source venv/bin/activate  # On Windows: venv\Scripts\activate
pip install -r requirements.txt
python app.py

# Ingestor (manual run)
cd medifinder-ingestor
python -m venv venv
source venv/bin/activate  # On Windows: venv\Scripts\activate
pip install -r requirements.txt
python ingest.py --source=sample_data.txt

# WhatsApp Bot (requires webhook configuration)
cd medifinder-bot
python -m venv venv
source venv/bin/activate  # On Windows: venv\Scripts\activate
pip install -r requirements.txt
python bot.py
```

See each repository's README for detailed component-specific instructions.

## 👥 Contributing

We welcome contributions to the MediFinderBot project! Please see our [Contributing Guidelines](CONTRIBUTING.md) for more information.

### Development Workflow
1. Create an issue or choose an existing one
2. Fork the repository and create a branch
3. Make your changes
4. Submit a pull request
5. Code review and merge

## 📄 License

This project is licensed under the MIT License - see the [LICENSE](LICENSE) file for details.

## 📞 Contact

For questions or support, please open an issue on this repository.
