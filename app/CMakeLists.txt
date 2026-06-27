target_include_directories(app PRIVATE include)

if(CONFIG_ZMK_STUDIO_RPC)
  add_subdirectory(src/studio)
endif()

target_sources_ifdef(CONFIG_ZMK_BEHAVIOR_STUDIO_UNLOCK app PRIVATE
                     src/behaviors/behavior_studio_unlock.c)
