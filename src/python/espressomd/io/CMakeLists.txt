add_subdirectory(writer)
SET(cython_AUX  ${cython_AUX} "${CMAKE_SOURCE_DIR}/src/python/espressomd/io/__init__.py" CACHE INTERNAL "cython_AUX" FORCE)
