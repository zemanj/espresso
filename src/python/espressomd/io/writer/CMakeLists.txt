configure_file(__init__.py __init__.py COPYONLY)
configure_file(vtf.py vtf.py COPYONLY)
if (HDF5_FOUND)
    configure_file(h5md.py h5md.py COPYONLY)
endif(HDF5_FOUND)
