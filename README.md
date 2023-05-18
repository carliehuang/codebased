# codebased: repository

view this file on github:
https://github.com/carliehuang/codebased/blob/main/README.md

## structure
`codebased/`
- `cmake/`
   - contains code coverage report configuration, used in CMakeLists.txt
- `docker/`
   - contains all files necessary to generate Docker image
- `include/`
   - contains all header files and canonical config files for both local testing and deployment
   - `http/`
      - contains all http-specific header files, i.e. specifying path, endpoint types
- `src/`
   - contains all source code files, implementations of classes
   - `http/`
      - contains all http-specific source files, i.e. extension mapping
- `tests/`
   - contains all testing files
   - `cases/`
      - contains all expected outputs for test cases
   - `static_files/`
      - contains all static files served, used in both test cases and deployment
- `CMakeLists.txt`
   - contains the commands necessary to build the project

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
1. `docker build -f docker/Dockerfile -t my_image .` to generate the Docker image from the docker file
2. `docker run --rm -p 8080:8080 --name my_run my_image:latest` to run the Docker image in a container locally
3. `docker container stop my_run` to stop the Docker container

## config file format
**include/config**
```
port 8080;

location /echo EchoHandler {
}

location /static1/ StaticHandler {
    root ../tests/static_files/static1;
}
location /static2/ StaticHandler {
    root ../tests/static_files/static2;
}
```
`port` keyword specifies port the server will run on

`location` keyword indicates path the handler with serve on

`<Type>Handler` indicates type of handler that will serve the specified path

`{}` specifies the arguments for a particular handler, empty if none

`root` keyword specifies filesystem path that handler will use

## request handlers
**include/request_handler_interface.h**
```
class request_handler_interface {
  public:
    virtual http::status serve(const http::request<http::dynamic_body> req, http::response<http::dynamic_body>& res) = 0;
    virtual ~request_handler_interface() {}
};
```
The request handler interface simply includes one function, serve, that all handlers must implement.

**include/echo_handler.h**
```
class echo_handler : public request_handler_interface {
  public:
    echo_handler(std::string location, std::string request_url);
    http::status serve(const http::request<http::dynamic_body> req, http::response<http::dynamic_body>& res);

  private:
    std::string location_;
    std::string request_url_;
    util utility;
};
```
Here, the echo handler class derives from the interface. It implements the serve function, as required, but also has a constructor with the necessary parameters and has member variables as necessary. The location string corresponds to the path after the location keyword in the config file and the request_url string corresponds with the actual text of the incoming request. These two will likely be in any new request handler, but there may be additional parameters as well. For example, in the static handler, the root file path is passed in as well.

**src/echo_handler.cc**
```
// Create basic echo handler with location
echo_handler::echo_handler(std::string location, std::string request_url)
  : location_(location), request_url_(request_url) {
}

// Construct echo reply
http::status echo_handler::serve(const http::request<http::dynamic_body> req, http::response<http::dynamic_body>& res){
  std::string input = req.target().to_string();
  if (request_url_ != input)
  {
    res.result(http::status::not_found);
    beast::ostream(res.body()) << utility.get_stock_reply(res.result_int());
    res.content_length((res.body().size()));
    res.set(http::field::content_type, "text/html");
    return res.result();
  }
  res.result(http::status::ok);
  beast::ostream(res.body()) << req;
  res.content_length((res.body().size()));
  res.set(http::field::content_type, "text/plain");
  return res.result();
}
```
In the implementation of the echo handler, the constructor assigns the parameters to the corresponding member ariables, and the serve function behaves according to the type of handler. In this case, the echo handler ensures that the request is valid and then simply sets the status, body, length, etc. of the http response. Other handlers may have more complicated implementations. For example, the static handler must determine if a file exists at a given path.

## request handler factories
**include/request_handler_factory.h**
```
// Parent class for request handler factory object
class request_handler_factory
{
  public:
    // creates a request handler by giving the config and path
    virtual request_handler_interface* create(NginxConfig config, std::string request_url_path) = 0;
    virtual ~request_handler_factory() {}
};
```
The base class for request handler factories simply includes one function, create, that derived classes must implement. The create function takes in an NginxConfig object, which contains the arguments {..} in the config file, and the request_url_path, which contains the actual request body, as specified in the Common API.

**include/static_handler_factory.h**
```
class static_handler_factory : public request_handler_factory
{
  public:
    static_handler_factory(std::string location);
    request_handler_interface* create(NginxConfig config, std::string request_url);
    std::string getPath(NginxConfig config);

  private:
    std::string location_;
    std::string request_url_;
    NginxConfig config_;
};
```
This snippet shows the static handler factory, which derives from request_handler_factory. It implements the create function and also contains a constructor and some member variables. Here, the location string corresponds to the location keyword in the config file— the path where handlers of this type serve.

**src/static_handler_factory.cc**
```
static_handler_factory::static_handler_factory(std::string location) :
    location_(location)
{
}

std::string static_handler_factory::getPath(NginxConfig config){
    std::string root = "";
    if (!config.statements_.empty() && config.statements_[0]->tokens_.size() == 2 && config.statements_[0]->tokens_[0] == "root") {
      // Remove double quoatations from root path
      root = config.statements_[0]->tokens_[1];
      size_t root_length = root.length();

    if (root_length > 0 && (root[0] == '"' || root[0] == '\'')) {
      root = root.erase(0, 1);
      root_length = root.length();
    }

    if (root_length > 0 && (root[root_length - 1] == '"' || root[root_length - 1] == '\'')) {
      root = root.erase(root_length - 1);
    }
  }
  return root;
}

request_handler_interface* static_handler_factory::create(NginxConfig config, std::string request_url){
  return new static_handler(location_, getPath(config), request_url);
}
```
In the implementation of the static handler factory, the constructor assigns the parameters to the corresponding member variables, and the create function returns a new static handler of the appropriate type. However, because static handlers need the filesystem path after the root keyword, the static handler factory uses a helper function, getPath, to parse the NginxConfig object and extract that path for the new static handler.

## steps to add a request handler and corresponding factory
1. Create header file `mynew_handler.h` in include/ folder, using the following as a template:
```
#ifndef MYNEW_HANDLER_H
#define MYNEW_HANDLER_H

#include "request_handler_interface.h"
#include <cstring>
// other files if necessary

class mynew_handler : public request_handler_interface{
    public:
        mynew_handler(std::string loc, std::string request_url); // other parameters if necessary
        http::status serve(const http::request<http::dynamic_body request, http::response<http::dynamic_body>& res);
        // other helper functions if necessary
    private:
        std::string loc_;
        std::string request_url_;
        // other member variables if necessary
};

#endif
```
2. Create header file `mynew_handler_factory.h` in include/ folder, using the following as a template:
```
class mynew_handler_factory : public request_handler_factory
{
  public:
    mynew_handler_factory(std::string location);
    mynew_handler_interface* create(NginxConfig config, std::string request_url);
    // other helper functions if necessary

  private:
    std::string location_;
    // other member variables if necessary
};
```
3. Create source file `mynew_handler.cc` in src/ folder, using the following as a template:
```
#include "mynew_handler.h"

mynew_handler::mynew_handler(std::string location, std::string request_url)
  : location_(location), request_url_(request_url) {
}
http::status mynew_handler::serve(const http::request<http::dynamic_body> req, http::response<http::dynamic_body>& res){
   // handler-specific implementation here
}
```
4. Create source file `mynew_handler_factory.cc` in src/ folder, using the following as a template:
```
#include "mynew_handler.h"
#include "mynew_handler_factory.h"

mynew_handler_factory::mynew_handler_factory(std::string location) :
      location_(location)
{
}

request_handler_interface* mynew_handler_factory::create(NginxConfig config, std::string request_url){
   // other handler-specific implementation, such as parsing config parameters, if necessary
  return new echo_handler(location_, request_url); //other parameters if necessary
}

// other helper function implementations if necessary
```
5. Include the new handler factory header file in `config_parser.cc`
6. Add condition to `config_parser.cc` GetLocationMap() function (snippet starts from line 83):
```
  if (handler == "StaticHandler") {
    factory_map_[handler] = new static_handler_factory(location);
  }
  if (handler == "EchoHandler") {
    factory_map_[handler] = new echo_handler_factory(location);
  }
  if (handler == MyNewHandler") {
    factory_map_[handler] = new mynew_handler_factory(location);
  }
  else factory_map_["ErrorHandler"] = new error404_handler_factory(location);
    }
  }
  return location_handlers;
```
7. Create file `tests/mynew_handler_test.cc` and write appropriate tests for the new handler
8. Add new handler to `CMakeLists.txt`:
```
add_library(mynew_handler src/mynew_handler.cc)
add_library(mynew_handler_factory src/mynew_handler_factory.cc)

// link mynew_handler and mynew_handler_factory to webserver executable on line 46

add_executable(mynew_handler_test tests/mynew_handler_test.cc)
target_link_libraries(mynew_handler_test mynew_handler Boost::log_setup Boost::log gtest_main)
gtest_discover_tests(mynew_handler_test WORKING_DIRECTORY ../tests)

// add mynew_handler and mynew_handler_factory libraries to coverage TARGETS and mynew_handler_test executable to coverage TESTS on line 95
```
9. Configure paths served by new handler by adding information to `include/config` and `include/config_google`:
```
port 8080;

location /echo EchoHandler {
}

location /static1/ StaticHandler {
    root ../tests/static_files/static1;
}

location /static2/ StaticHandler {
    root ../tests/static_files/static2;
}

location /pathtoserve/ MyNewHandler {
     // parameters if any
}
```

