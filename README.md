# ngab0016 - 041196196
# 1.Demo video
- Here is the unlisted Youtube Video: https://youtu.be/rC5SdROLeYg

# 2. Brief Technical explanation
 ## Order Service (Node.js)

 Order Service is responsible for handling order requests from clients. It is called by the front-end when an order is submitted by a user, handles order data (product, count, total price) and can forward it to other systems like RabbitMQ. It is implemented with Node.js programming language, typically with HTTP routes with Express.js. Its purpose is to expose an API endpoint (e.g., POST /orders) to be called by the front-end(store-front) implemented with Vue.js. It is a bridge between UI and backend processing logic about ordering so that data is routed to the right place to be processed.

 ## Product Service (Rust + Warp)

 Product Service is charged with managing the product catalog. In this case, it has a simple REST API (via Warp framework in Rust) with an endpoint like GET /products that serves a list of available pet products as JSON. It is a lightweight backend microservice & is called by the Vue front-end to display an option list of products to customers. On a much larger scale, it would also pull real product data out of a database but for purposes of simplification it serves static JSON. It's most critical function is to offer up the product data that Order Service and Store Front depend on.

 ## Store Front(Vue.js)
 Store Front is a client-side(Front-end) application where customers interact with the pet store. It is implemented using a newer front-end framework called Vue.js. It is a service that presents the UI, so that users can navigate products, select quantities, and order. It communicates directly with the Product Service using REST API calls to view product details, and with the Order Service to add new orders. It is a presentation layer that binds the backend services together to turn it into a logical and valid user experience
