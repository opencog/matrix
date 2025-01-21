
# The atom_types.h file is written to the build directory
INCLUDE_DIRECTORIES(${CMAKE_BINARY_DIR})

# DECLARE_GUILE_CONFIG_TARGET(SCM_CONFIG "opencog as-matrix-config" "ATOMSPACE_TEST")

ADD_SUBDIRECTORY (matrix)

# WRITE_GUILE_CONFIG(${GUILE_BIN_DIR}/opencog/as-matrix-config.scm SCM_CONFIG TRUE)
#
# WRITE_GUILE_CONFIG(${GUILE_BIN_DIR}/opencog/as-matrix-config-installable.scm SCM_CONFIG FALSE)
# INSTALL(FILES ${GUILE_BIN_DIR}/opencog/as-matrix-config-installable.scm DESTINATION ${GUILE_SITE_DIR}/opencog RENAME as-matrix-config.scm)
