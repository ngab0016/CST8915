# ngab0016, CST8915

## 1. Demo video

- Youtube video link(unlisted): https://youtu.be/GM5z3aKx6wU

## 2. Reflection questions

i. What challenges did you encounter when configuring environment variables in the GitHub Actions workflow?

- One of the important challenge was that all of the environment variables should be set rightly both on GitHub Actions and on Azure App Service. Additionally, I should ensure that the requirements.txt within the product-service contained the python-dotenv package since without it the Azure app continuously failed with an “Application Error.” Secure handling of secrets, proper referencing of the secrets within the workflow, and aligning the local dev env with that of the cloud env were all important considerations to achieve getting the deployment to run smoothly.

ii. How does deploying microservices on Azure Web App Service differ from running them locally?

- Running microservices on Azure Web App Service is unique to operating locally in that Azure offers a managed setup whereby the dependencies get installed automatically, network setup occurs automatically and scaling is supported. Locally, I would have to install packages manually and set up the environment. Personally, I actually prefer to run the app on Azure because deployment is made easier & the dependencies get taken care of by you and you get to have stability of production without having to set up all the infrastructure locally.

iii. Why is it important to use environment variables for configurations in a cloud environment?

- It's valuable to employ environment variables within a cloud environment because it keeps sensitive data such as API keys or database credentials out of the source code which enhances security. It enables the codebase to run the same codebase across various environments—development, staging and production without any modification to the code itself. Environment variables make configuration flexible which allows you to make configuration alterations without redeploying the application and they aid best practices of cloud-native applications.

## 3. Links to repositories:

- Order-service: https://github.com/ngab0016/order-service.git
- Product-service: https://github.com/ngab0016/Product-service.git
- Store-front: https://github.com/ngab0016/store-front.git