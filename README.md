# go-mistral

A Go client library for accessing the [Mistral AI API](https://docs.mistral.ai/).

## Installation

```bash
go get github.com/tforrest/mistral-api-go
```

## Usage

```go
import "github.com/tforrest/mistral-api-go"

// Create a new client
client, err := mistral.NewClient("your-api-key")
if err != nil {
    log.Fatal(err)
}

// Create a chat completion
ctx := context.Background()
resp, err := client.Chat.Create(ctx, &mistral.ChatCompletionRequest{
    Model: "mistral-small-latest",
    Messages: []mistral.Message{
        {
            Role:    "user",
            Content: "What is the capital of France?",
        },
    },
})
if err != nil {
    log.Fatal(err)
}
fmt.Println(resp.Choices[0].Message.Content)

// Stream chat completions
req := &mistral.ChatCompletionRequest{
    Model: "mistral-small-latest",
    Messages: []mistral.Message{
        {
            Role:    "user",
            Content: "Write a story about a space adventure.",
        },
    },
    Stream: true,
}

stream, err := client.Chat.CreateStream(ctx, req)
if err != nil {
    log.Fatal(err)
}
defer stream.Close()

for {
    response, err := stream.Recv()
    if err == io.EOF {
        break
    }
    if err != nil {
        log.Fatal(err)
    }
    fmt.Print(response.Choices[0].Message.Content)
}

// Generate embeddings
embedResp, err := client.Embeddings.Create(ctx, &mistral.EmbeddingRequest{
    Model: "mistral-embed",
    Input: []string{"Hello, world!"},
})
if err != nil {
    log.Fatal(err)
}
fmt.Println(embedResp.Data[0].Embedding)

// List available models
models, err := client.Models.List(ctx)
if err != nil {
    log.Fatal(err)
}
for _, model := range models.Data {
    fmt.Println(model.ID)
}

// Perform OCR on images
files := []string{"file1_id", "file2_id"} // File IDs from previously uploaded images
ocrResp, err := client.OCR.Create(ctx, &mistral.OCRRequest{
    Model: "mistral-ocr-v1",
    Files: files,
    Languages: []string{"en", "fr"}, // Optional: specify languages to detect
})
if err != nil {
    log.Fatal(err)
}
for _, result := range ocrResp.Results {
    fmt.Printf("Text from %s: %s\n", result.FileID, result.Text)
    fmt.Printf("Detected language: %s\n", result.Language)
    for _, block := range result.Blocks {
        fmt.Printf("Block text: %s (confidence: %.2f)\n", block.Text, block.Confidence)
    }
}

// For large documents, use async OCR
jobID, err := client.OCR.CreateAsync(ctx, &mistral.OCRRequest{
    Model: "mistral-ocr-v1",
    Files: files,
})
if err != nil {
    log.Fatal(err)
}

// Poll for results
result, err := client.OCR.GetAsyncResult(ctx, jobID)
if err != nil {
    log.Fatal(err)
}
fmt.Printf("Async OCR completed with %d results\n", len(result.Results))
```

## Features

- Complete Mistral AI API coverage
- Streaming support for chat completions and agents
- Context support for cancellation and timeouts
- Configurable HTTP client
- Custom base URL support
- Error handling and response parsing
- File upload and download support
- Fine-tuning job management
- Content moderation
- Optical Character Recognition (OCR)
- Agent-based interactions with custom tools
- Advanced embedding operations with similarity calculation
- Batch processing for embeddings
- Fill-in-the-Middle (FIM) text generation
- Text classification with multi-label support
- Enhanced model information and capabilities
- Token estimation and model performance metrics
- Batch processing with retry and concurrency control

## Services

The client is divided into several services:

- `Chat`: Chat completions
- `Models`: Model management with version tracking and performance metrics
- `Embeddings`: Text embeddings with similarity and batch processing
- `Files`: File operations
- `FineTuning`: Fine-tuning jobs
- `Moderations`: Content moderation
- `OCR`: Optical Character Recognition
- `Agents`: AI agents with custom tool support
- `FIM`: Fill-in-the-Middle text generation
- `Classifiers`: Text classification with multi-label support
- `Batch`: Efficient batch processing of requests

## Configuration

You can configure the client using options:

```go
client, err := mistral.NewClient(
    "your-api-key",
    mistral.WithBaseURL("https://custom-url.com"),
    mistral.WithHTTPClient(&http.Client{
        Timeout: time.Second * 30,
    }),
)
```

## Error Handling

The library returns detailed error messages from the API:

```go
_, err := client.Chat.Create(ctx, req)
if err != nil {
    if apiErr, ok := err.(*mistral.Error); ok {
        fmt.Printf("API error: %v\n", apiErr.Message)
        fmt.Printf("Status code: %d\n", apiErr.Response.StatusCode)
    }
    return err
}
```

## Usage Examples

```go
// Fill-in-the-Middle example
fimResp, err := client.FIM.Create(ctx, &mistral.FIMRequest{
    Model:  "mistral-large-latest",
    Prefix: "The quick brown",
    Suffix: "jumped over the lazy dog",
})
if err != nil {
    log.Fatal(err)
}
fmt.Println(fimResp.Choices[0].Text)

// Text classification example
classResp, err := client.Classifiers.Create(ctx, &mistral.ClassifierRequest{
    Model:  "mistral-classify",
    Input:  []string{"This movie was fantastic!"},
    Labels: []string{"positive", "negative", "neutral"},
})
if err != nil {
    log.Fatal(err)
}
fmt.Printf("Classification: %s (confidence: %.2f)\n",
    classResp.Results[0].Labels[0],
    classResp.Results[0].Confidence)

// Enhanced model information
modelInfo, err := client.Models.GetEnhanced(ctx, "mistral-large-latest")
if err != nil {
    log.Fatal(err)
}
fmt.Printf("Model version: %s\n", modelInfo.Version)
fmt.Printf("Max tokens: %d\n", modelInfo.MaxTokens)
for _, cap := range modelInfo.Capabilities {
    fmt.Printf("Capability: %s (%v)\n", cap.Name, cap.Available)
}

// Token estimation
tokenCount, err := client.Models.EstimateTokens(ctx, "mistral-large-latest",
    "How many tokens are in this text?")
if err != nil {
    log.Fatal(err)
}
fmt.Printf("Token count: %d\n", tokenCount)

// Batch processing example
texts := []string{"Text 1", "Text 2", "Text 3", /* ... */}
batchResp, err := client.Batch.CreateEmbeddings(ctx, texts, "mistral-embed", &mistral.BatchOptions{
    MaxConcurrency: 5,
    ChunkSize:      1000,
    RetryConfig: &mistral.Retry{
        MaxAttempts:  3,
        InitialDelay: 1000,
        MaxDelay:     5000,
    },
})
if err != nil {
    log.Fatal(err)
}

// Wait for batch completion
result, err := client.Batch.WaitForCompletion(ctx, batchResp.ID)
if err != nil {
    log.Fatal(err)
}
fmt.Printf("Processed %d/%d requests successfully\n",
    result.Summary.Succeeded,
    result.Summary.TotalRequests)
```

## Contributing

Contributions are welcome! Please feel free to submit a Pull Request.

## License

This library is distributed under the MIT license.
