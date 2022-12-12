#-----------------------------------------------------------------------------
add_executable(reg_average reg_average.cpp)
target_link_libraries(reg_average _reg_globalTransformation _reg_maths _reg_tools _reg_ReadWriteImage)
#-----------------------------------------------------------------------------
add_executable(reg_tools reg_tools.cpp)
target_link_libraries(reg_tools _reg_resampling _reg_localTransformation _reg_tools _reg_globalTransformation _reg_ReadWriteImage)
#-----------------------------------------------------------------------------
add_executable(reg_resample reg_resample.cpp)
target_link_libraries(reg_resample _reg_resampling _reg_localTransformation _reg_tools _reg_globalTransformation _reg_ReadWriteImage)
#-----------------------------------------------------------------------------
add_executable(reg_transform reg_transform.cpp)
target_link_libraries(reg_transform _reg_thinPlateSpline _reg_resampling _reg_localTransformation _reg_tools _reg_globalTransformation _reg_maths _reg_ReadWriteImage)
#-----------------------------------------------------------------------------
add_executable(reg_jacobian reg_jacobian.cpp)
target_link_libraries(reg_jacobian _reg_resampling _reg_localTransformation _reg_tools _reg_globalTransformation _reg_ReadWriteImage)
#-----------------------------------------------------------------------------
if(USE_CUDA)
        add_executable(reg_f3d reg_f3d.cpp)
else(USE_CUDA)
        add_executable(reg_f3d reg_f3d.cpp)
endif(USE_CUDA)
target_link_libraries(reg_f3d _reg_f3d cuda)