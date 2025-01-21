
# CMake boilerplate that allows users to do
#    find_package(AsMatrix REQUIRED 2.0.0)
# and have it work.

include(CMakePackageConfigHelpers)

set(ConfigPackageLocation lib/cmake/AsMatrix)

#install(EXPORT AsMatrixTargets
#	FILE AsMatrixTargets.cmake
#	DESTINATION ${ConfigPackageLocation}
#)

SET(SEMANTIC_VERSION 2.0.0)

configure_package_config_file(AsMatrixConfig.cmake.in
	${CMAKE_CURRENT_BINARY_DIR}/AsMatrixConfig.cmake
	INSTALL_DESTINATION ${ConfigPackageLocation}
	PATH_VARS CMAKE_INSTALL_PREFIX
)

write_basic_package_version_file(
	"${CMAKE_CURRENT_BINARY_DIR}/AsMatrixConfigVersion.cmake"
	VERSION ${SEMANTIC_VERSION}
	COMPATIBILITY SameMajorVersion
)

INSTALL(FILES
	${CMAKE_CURRENT_BINARY_DIR}/AsMatrixConfigVersion.cmake
	${CMAKE_CURRENT_BINARY_DIR}/AsMatrixConfig.cmake
	DESTINATION ${ConfigPackageLocation}
)

# -----------------------------------------------------------
