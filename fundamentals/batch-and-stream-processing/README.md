# Batch & Stream Processing

MapReduce, Spark, Lambda architecture, stream processing theory, data pipeline orchestration.

## Sub-topics

- **[Batch Processing](batch-processing.md)** ✅: MapReduce, Spark, Hadoop
- **[Lambda Architecture](lambda-architecture.md)** ✅: Batch + stream combined
- **[Kappa Architecture](kappa-architecture.md)** ✅: Stream-only design, event log replay, vs Lambda tradeoffs
- **[Stream Processing Theory](stream-processing-theory.md)** ✅: Windowing, watermarks, triggering, state management
- **[Stream Processing (Frameworks)](../messaging-and-streaming/stream-processing.md)** ✅ (see Messaging: Flink, Spark)

## Why This Matters

- **Scale**: Process petabytes of data in batch or real-time
- **Latency trade-off**: Batch (hours, cheaper) vs stream (minutes, more expensive)
- **Architecture choice**: Lambda (dual) vs Kappa (stream-only) affects operational complexity
- **Production patterns**: Spark for analytics, Flink for real-time, event sourcing for replay

## Interview Prep

- **[Question Bank](../../interview-prep/question-banks/batch-and-stream-processing.md)**: 25+ questions on MapReduce, Spark, Kappa/Lambda
- **[Flashcards](../../interview-prep/flashcards/batch-and-stream-processing.md)**: Windowing, state management, exactly-once

## Related Fundamentals

- [Messaging & Streaming](../messaging-and-streaming/) – Kafka, event sourcing, stream processing frameworks
- [Databases](../databases/) – Data storage, replication, partitioning

## Status

✅ **COMPLETE**. 4/4 files written (Batch, Lambda, Kappa, Stream Theory).

