# Introducing [mongoose-ai](https://www.npmjs.com/package/@jmndao/mongoose-ai): AI-Powered Document Processing for MongoDB

While building an AI agent that required insights on MongoDB data, I encountered a critical issue: my tokens were being consumed at an alarming rate. The agent endpoint was fetching entire documents and sending them to OpenAI for processing, but this approach was inefficient and expensive. I realized I needed a more intelligent way to process documents that would optimize token usage while maintaining AI capabilities. That's why I created [mongoose-ai](https://www.npmjs.com/package/@jmndao/mongoose-ai)—a Mongoose plugin that enables smarter, more cost-effective AI processing directly within MongoDB workflows.

## The Problem We're Solving

AI agents working with MongoDB data face a token efficiency challenge: sending entire documents to language models consumes tokens rapidly and drives up costs. Without intelligent field selection and processing optimization, even modest document collections can exhaust token budgets quickly, making AI-powered features unsustainable for production use.

## The mongoose-ai Solution

[mongoose-ai](https://www.npmjs.com/package/@jmndao/mongoose-ai) eliminates token waste by providing intelligent document processing that works automatically within your existing Mongoose schemas:

**Smart Field Processing**: Only processes specified fields instead of entire documents, dramatically reducing token consumption.

**Automatic Summarization**: Documents receive AI-generated summaries optimized for token efficiency, without manual intervention.

**Semantic Search**: Query your data using natural language while maintaining cost-effective token usage patterns.

## Quick Start

Getting started requires minimal setup and no breaking changes to existing schemas.

### Installation

```bash
npm install @jmndao/mongoose-ai
```

[**Get started with the project on GitHub →**](https://github.com/jmndao/mongoose-ai)

### Basic Implementation

```javascript
import mongoose from "mongoose";
import { aiPlugin } from "@jmndao/mongoose-ai";

const articleSchema = new mongoose.Schema({
  title: String,
  content: String,
  author: String,
});

// Enable automatic AI summarization
articleSchema.plugin(aiPlugin, {
  ai: {
    model: "summary",
    provider: "openai",
    field: "aiSummary",
    credentials: {
      apiKey: process.env.OPENAI_API_KEY,
    },
  },
});

const Article = mongoose.model("Article", articleSchema);

// AI summaries are generated automatically on save
const article = new Article({
  title: "Introduction to Machine Learning",
  content:
    "Machine learning is a subset of artificial intelligence that enables computers to learn and improve from experience without being explicitly programmed...",
});

await article.save();
console.log(article.aiSummary.summary); // AI-generated summary available immediately
```

### Semantic Search Setup

```javascript
// Enable semantic search capabilities
productSchema.plugin(aiPlugin, {
  ai: {
    model: "embedding",
    provider: "openai",
    field: "searchEmbedding",
    credentials: {
      apiKey: process.env.OPENAI_API_KEY,
    },
  },
});

// Search using natural language queries
const results = await Product.semanticSearch(
  "wireless headphones with noise cancellation",
  {
    limit: 5,
    threshold: 0.7,
  }
);
```

## Performance and Cost Analysis

Extensive production testing reveals compelling performance metrics:

### Processing Performance

- **Processing time**: 1,558ms per document
- **Throughput**: 38.5 documents per minute
- **Memory usage**: 2-5MB per document during processing

### Token Efficiency and Cost Savings

- **AI processing cost**: $0.000343 per document (optimized field processing)
- **Token reduction**: Up to 80% fewer tokens compared to full document processing
- **Smart field selection**: Process only relevant fields, not entire documents
- **Sustainable scaling**: Cost-effective processing even for large document collections

The plugin's intelligent field processing dramatically reduces token consumption, making AI-powered features sustainable for production use while maintaining processing quality.

## Advanced Configuration

The plugin supports extensive customization for production environments:

```javascript
import { createAIConfig } from "@jmndao/mongoose-ai";

schema.plugin(aiPlugin, {
  ai: createAIConfig({
    apiKey: process.env.OPENAI_API_KEY,
    model: "summary",
    field: "aiInsights",
    prompt:
      "Create a professional summary highlighting key insights and actionable takeaways:",

    // Control which fields are processed to optimize token usage
    includeFields: ["title", "content", "category"],
    excludeFields: ["author", "metadata", "timestamps"],

    // Performance and reliability settings
    advanced: {
      maxRetries: 3,
      timeout: 45000,
      skipOnUpdate: true,
      logLevel: "info",
    },

    // Fine-tune AI model behavior
    modelConfig: {
      chatModel: "gpt-4",
      maxTokens: 150,
      temperature: 0.2,
    },
  }),
});
```

## Technical Architecture

[mongoose-ai](https://www.npmjs.com/package/@jmndao/mongoose-ai) follows clean architecture principles that ensure maintainability and extensibility:

**Provider Abstraction**: Current OpenAI support with interfaces designed for easy integration of additional providers (Anthropic Claude, Google Bard, etc.).

**Schema Integration**: Dynamically enhances existing schemas without breaking changes or requiring migration.

**Error Resilience**: Robust error handling ensures document saves succeed even when AI processing encounters issues.

**Type Safety**: Comprehensive TypeScript support with full type definitions.

**Independent Processing**: Documents are processed independently, providing clear scaling paths for large datasets.

## Production Considerations

### Scaling Strategy

The current implementation efficiently handles up to 10,000 documents. For larger datasets, consider integrating vector databases or MongoDB Atlas Vector Search for optimal semantic search performance.

### Token Optimization

The plugin's intelligent field processing is the key to sustainable AI integration. By processing only specified fields rather than entire documents, token consumption is dramatically reduced while maintaining processing quality.

### Security and Reliability

- API credentials are handled securely without storing data outside your infrastructure
- Built-in cost monitoring and token counting help manage expenses
- Documents save successfully even if AI processing fails, ensuring application reliability

### Cost Management

Built-in token counting and cost estimation help monitor expenses and optimize field selection for maximum efficiency. The plugin includes utilities for cost projection and token usage analysis.

## Real-World Applications

[mongoose-ai](https://www.npmjs.com/package/@jmndao/mongoose-ai) excels in diverse scenarios:

- **Content Management Systems**: Automated article summarization and intelligent content discovery
- **Knowledge Bases**: Enhanced search capabilities for documentation and support materials
- **E-commerce Platforms**: Product description processing and semantic product search
- **Document Repositories**: Intelligent organization and retrieval of business documents

## Getting Started

[mongoose-ai](https://www.npmjs.com/package/@jmndao/mongoose-ai) is available through standard JavaScript package managers:

- **npm Package**: https://www.npmjs.com/package/@jmndao/mongoose-ai
- **GitHub Repository**: https://github.com/jmndao/mongoose-ai

[**Start building with mongoose-ai on GitHub →**](https://github.com/jmndao/mongoose-ai)

The repository includes comprehensive documentation, practical examples, and benchmark scripts. Docker support enables immediate testing without local environment setup.

## Contributing and Future Development

This open-source project welcomes community contributions. The modular architecture supports adding new AI providers through clean interfaces. Current development priorities include:

- Additional AI provider integrations
- Enhanced vector database support
- Advanced filtering and processing capabilities
- Performance optimizations for large-scale deployments

## Conclusion

[mongoose-ai](https://www.npmjs.com/package/@jmndao/mongoose-ai) solves the critical token efficiency problem that makes AI agents expensive to operate. By enabling intelligent field processing and optimizing token usage, developers can build sustainable AI-powered features without worrying about runaway costs.

The plugin's smart processing approach creates new possibilities for cost-effective AI integration while maintaining the simplicity that makes MongoDB and Mongoose popular choices for modern applications. For teams building AI agents that work with document data, the dramatic reduction in token consumption makes [mongoose-ai](https://www.npmjs.com/package/@jmndao/mongoose-ai) an essential tool for sustainable development.

Transform your AI agent from a token-burning liability into a cost-efficient asset—experience optimized AI document processing today.
