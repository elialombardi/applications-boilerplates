write a complete prompt to build new web applications. It must specifiy folder structure and code examples. The result must have a backend and a frontend folder. Backend must be structured with a folder for a web api, a console application and a library which will be used for shared logic, like database models, shared business logic, ecc.

Backend must be written in go. It must use fiber v3. It must be structured in layers: service layer to interact with database. Controller layer and console application workers must provide DTO to service layer which will be then used to interact with database. Service layer will have business logic. Database will be postgresq and it must use an ORM.

Frontend must use react, RTK Query and material ui dark theme.

The application must use JWT for authentication and it must have registration page, login page and a main page to land after login.

Have an initial section for placeholders where I can specify the application name, focus and features. 

Have a section with the initial cli commands for configuring folders, install packages and run debug.