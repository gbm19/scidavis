if( ENABLE_Testing )

  message( STATUS "Testing enabled" )

  set( CMAKE_AUTOMOC OFF )
  set( CMAKE_AUTOUIC OFF )
  set( CMAKE_AUTORCC OFF )

  set( SRCS
    "main.cpp"
    "applicationWindow.cpp"
    "readWriteProject.cpp"
    "fft.cpp"
    "menus.cpp"
    "arrowMarker.cpp"
    )
  if( NOT WIN32 )
    list( APPEND SRCS
      "testPaintDevice.cpp"
      "3dplot.cpp"
      )
  endif()

  add_executable( unittest ${SRCS} )

  # Find UnitTest++ library and its headers
  find_package( UnitTest++ CONFIG )
  if( UnitTest++_FOUND )
    target_link_libraries( unittest UnitTest++ )
    target_include_directories( unittest PRIVATE ${UTPP_INCLUDE_DIRS} )
  else()
    find_package( PkgConfig )
    if( PKG_CONFIG_FOUND )
      pkg_search_module( UnitTest UnitTest++ IMPORTED_TARGET )
      if( UnitTest_FOUND )
        target_link_libraries( unittest PkgConfig::UnitTest )
      endif()
    endif()
    
    if( NOT PKG_CONFIG_FOUND OR NOT UnitTest_FOUND )
      find_library( UNITTEST_LIBRARY UnitTest++ REQUIRED )
      find_path ( UNITTEST_INCLUDE_DIR
        NAMES UnitTest++.h
        PATH_SUFFIXES UnitTest++
        REQUIRED
        )
      target_link_libraries( unittest ${UNITTEST_LIBRARY} )
      target_include_directories( unittest PRIVATE ${UNITTEST_INCLUDE_DIR} )
    endif()
  endif()
  # Find Boost-regex library
  find_package(Boost REQUIRED COMPONENTS regex)

  target_link_libraries( unittest libscidavis Boost::regex )
  if( NOT WIN32 )
    target_link_libraries( unittest pthread )
  endif()

  if( MULTI_CONFIG )
    target_include_directories( unittest PRIVATE
      "${CMAKE_BINARY_DIR}/libscidavis/libscidavis_autogen/include_$<CONFIG>"
      )
  else()
    target_include_directories( unittest PRIVATE
      "${CMAKE_BINARY_DIR}/libscidavis/libscidavis_autogen/include"
      )
  endif()

  if( ORIGIN_IMPORT )
    target_compile_definitions( unittest PRIVATE ORIGIN_IMPORT )
  endif()

  add_test( NAME unittest COMMAND unittest )

  set( TestDataFiles
    "3dplot.sciprj"
    "3dplotTable.sciprj"
    "testProject.sciprj"
    "Histo.opj"
    "USstates.opj"
    )
  file( COPY ${TestDataFiles} DESTINATION . )

  if( NOT (SEARCH_FOR_UPDATES OR DOWNLOAD_LINKS) )

    set( menu_files defaultMenus_en.menudat appWithGraphen.menudat )

    foreach( menu ${menu_files} )
      get_filename_component( test_name ${menu} NAME_WLE )
      add_test( NAME menuData_${test_name} COMMAND diff -q ${menu} ${CMAKE_CURRENT_SOURCE_DIR}/menuData/${menu} )
      set_tests_properties( menuData_${test_name} PROPERTIES DEPENDS unittest )
    endforeach()

  endif()

  if( SCRIPTING_PYTHON )

    file( COPY pythonTests/ DESTINATION ./tmp )
    file( COPY ../scidavis-logo.png DESTINATION ./tmp )

    file( GLOB pythonTests pythonTests/*.py )
    # qwtPlotCurve segfault with sip5 or macos
    if( SIP_VERSION VERSION_GREATER_EQUAL 5 OR APPLE )
      list( REMOVE_ITEM pythonTests ${CMAKE_CURRENT_SOURCE_DIR}/pythonTests/qwtPlotCurve.py )
    endif()

    foreach( pythonTest ${pythonTests} )
      get_filename_component( test_name ${pythonTest} NAME_WLE )
      add_test( NAME python_${test_name}
        COMMAND scidavis -l=en -x ${pythonTest}
        WORKING_DIRECTORY ${CMAKE_CURRENT_BINARY_DIR}/tmp
      )
    endforeach()

  endif()

else( ENABLE_Testing )

  message( STATUS "Testing not enabled" )

endif( ENABLE_Testing )
