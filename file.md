useEffect(() => {
  if (active) {
    fetchFiles({ page: 1 });
  } else {
    skipNextSearchEffect.current = true;
    setSearchQuery('');
    dispatch(setStatusFilter(''));
    dispatch(serverFilesSuccess({
      files: [], total: 0, page: 1, pageSize: filesPageSize, totalPages: 1,
      sortBy: 'id', sortOrder: 'desc', search: '',
    }));
  }
}, [active]);
