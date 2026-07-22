# New Concepts

| Concept | What is it? | Short example |
|---|---|---|
| **Elasticsearch** | A fast search and analytics engine used to search large volumes of text and logs. | Searching for hotels by name or city and analyzing application logs. |
| **Orchestrator** | A system that coordinates, runs, monitors, and scales multiple services or tasks. | **Kubernetes** manages application containers and restarts a container if it fails. |
| **TTL (Time To Live)** | The amount of time data remains valid before it is automatically deleted. It is commonly used for temporary data in **Redis**. | `SET booking:123 "pending" EX 300` stores the booking for 5 minutes. |

## Problems to Analyze from the Beginning

Based on the system design, the main areas to analyze are:

1. Multi-provider integration
2. Provider adapters
3. Response normalization
4. Parallel provider search
5. Partial failure handling
6. Search caching in Redis
7. Offer expiration
8. Price revalidation
9. Deduplication and ranking
10. Provider rate limits
11. Booking orchestration
12. Payment and booking consistency
13. Provider webhooks
14. Failover and retries
15. Observability per provider
