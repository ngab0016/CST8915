# ngab0016 - 041196196

## Demo video

- Youtube video link: https://youtu.be/BQWoU3R0sBI

## Written analysis on Rabbit MQ
### Is RabbitMQ Stateless or Stateful?
RabbitMQ is a stateful application. It maintains critical data such as  Message queues and their contents, Exchange configurations, Binding rules, User credentials and permissions & Message persistence and acknowledgments

### Running Without Persistent Storage

What Happens:
When a RabbitMQ pod is deleted or restarted without persistent storage, all data is lost because container filesystem is ephemeral, Data exists only in memory and container storage
 and therefore Pod recreation = brand new container = empty state.

Implications of this problem:
You lose customer orders, Inconsistent application state, failed message delivery, you need to manually reconfigure RabbitMQ as well as potential data corruption in dependent services.

### Solutions

- Use Azure Service Bus for fully managed messaging service & built-in high availability
- Use Kubernetes PersistentVolume to attach external storage to the pod & data survives pod restarts
- StatefulSets which are better for stateful applications & stable network identities

### Does Azure Service Bus Solve the RabbitMQ Problem?

Yes, it does as no pod management required, data is automatically persisted and replicated, built-in disaster recovery & Higher availability guarantees. However, it equires code changes (different API) & has a monthly cost
