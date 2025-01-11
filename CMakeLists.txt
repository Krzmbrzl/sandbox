cmake_minimum_required(VERSION 3.28 FATAL_ERROR)

project(
	sandbox
	VERSION 1.0.0
	LANGUAGES CXX
)

set(DIRS_TO_BUILD "modules" CACHE STRING "List of directories to include when building this project")

foreach(current_dir IN LISTS DIRS_TO_BUILD)
	add_subdirectory("${current_dir}")
endforeach()
