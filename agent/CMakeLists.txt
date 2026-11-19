cmake_minimum_required(VERSION 3.21)
project(bibliotheque)


option(BUILD_PYTHON_BINDINGS "Construire le module Python pybind11" OFF)

add_library(bibliotheque_core STATIC
book.cpp
user.cpp
library.cpp
filemanager.cpp
)

target_include_directories(bibliotheque_core PUBLIC ${CMAKE_CURRENT_SOURCE_DIR})
target_compile_options(bibliotheque_core PRIVATE -Wall -Wextra -Wpedantic)

#CLI seulement dans main.cpp
add_executable(bibliotheque main.cpp)
target_link_libraries(bibliotheque PRIVATE bibliotheque_core)



# flag pour tous les warnings possible
target_compile_options(${PROJECT_NAME} PRIVATE -Wall -Wextra -Wpedantic)
