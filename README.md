# codebased: repository

## structure
`codebased/` root directory
- `cmake/` contains code coverage report configuration, used in CMakeLists.txt
- `docker/` contains all files necessary to generate Docker image
- `include/` contains all header files and canonical config files for both local testing and deployment
   - `http/` contains all header files for 
- `src/` contains all source code files, implementations of classes
   - `http/` contains 
- `tests/` contains all testing files
   - `cases/` contains all expected outputs for test cases
   - `static_files/` contains all static files served, used in both test cases and deployment
- `CMakeLists.txt` contains the commands necessary to build the project

## build
commands to perform an out-of-source build in unix terminal:
1. `mkdir build` to create build directory
2. `cd build` to enter the new build directory
3. `cmake ..` to generate a Makefile
4. `make` to build the program using the generated Makefile

## test
commands to test the built program *in* the build directory using commands:
1. `make test` to run unit and integration tests
2. `ctest -V` to run unit and integration tests with more detailed output

## run
commands to run the webserver locally, starting in the build directory:
1. `cd bin` to enter the directory with executables
2. `./webserver ../../include/config` to start the server with the canonical config

## docker
commands to generate Docker image and run container locally:
1. Here
2. Here

## config file format
**include/config**
```
port 8080;

echo {
    location /echo{
    }
}

static {
    location /static1/ {
        root ../tests/static_files/static1;
    }

    location /static2/ {
        root ../tests/static_files/static2;
    }
}
```
`port` keyword specifies port the server will run on

`echo` keyword indicates bracketed information should be handled by echo handlers

`location` keyword indicates path

`static` keyword 

`root` keyword

## request handler interface
**include/request_handler_interface.h**
```
class request_handler_interface {
  public:
    virtual reply get_reply() = 0;
    // this will be the function that will take over get_reply after migrating to the updated API; error404 already uses this
    // but is not being used yet because the other handlers have not switched over yet. This will be uncommented once that 
    // change happens
    // virtual http::status serve(const http::request<http::dynamic_body> req, http::response<http::dynamic_body>& res) = 0;
    virtual ~request_handler_interface() {}
};
```
explanation of request handler interface

**src/static_handler.cc**
```
error404_handler::error404_handler(std::string loc, std::string request_url){
    std::string loc_ = loc;
    std::string request_url_ = request_url;
}
http::status error404_handler::serve(const http::request<http::dynamic_body> request, http::response<http::dynamic_body>& res){
    res.result(http::status::not_found);
    beast::ostream(res.body()) << utility.get_stock_reply(res.init());
    res.set(http::field::content_type, "text/html");
    return http::status::not_found;
}
```
explanation of child handler

## add another request handler
1. Create header file `mynew_handler.h` in include/ folder, using the following as a template:
```
#ifndef MYNEW_HANDLER_H
#define MYNEW_HANDLER_H

#include "request_handler_interface.h"
#include <cstring>

class mynew_handler : public request_handler_interface{
    public:
        mynew_handler(std::string loc, std::string request_url);
        http::status serve(const http::request<http::dynamic_body request, http::response<http::dynamic_body>& res);
    private:
        std::string loc_;
        std::string request_url_;
        util utility;
};

#endif
```
2. Create source file `mynew_handler.cc` in src/ folder, using the following as a template:
3. Configure paths served by new handler by adding information to `include/config` and `include/config_google`:
4. Add new endpoints to `include/http/paths.h`
5. Parse new endpoints in `src/config_parser.cc`
6. Add new handler to `CMakeLists.txt`
