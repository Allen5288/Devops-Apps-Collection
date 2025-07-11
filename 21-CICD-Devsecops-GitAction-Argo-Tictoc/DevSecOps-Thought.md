# DevSecOps Thinking

why devsecops?

- AI write code lead to code with hard code, unsecurity package, apiToken,
- Developer for some logic, some hard code from dev to prod; also some logged to public,
- secber secuity

Implement process;

- understand the project with developer before cicd
- understand the npm for package and vite for running
- npm install - download dependence
- npm run build - build application
- Dist folder
- Copy Dist to nginx

Docker:

- docker build -t tiktaktoe-demo:v1 .
- docker run -d -p 9099:80 tiktaktoe-demo:v1
