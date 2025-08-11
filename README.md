.sort-indicator {
  display: flex;
  justify-content: space-between; /* keep Sorted by on left, count on right */
  align-items: center;
  width: 100%;
  margin: 8px 0 0 0;
  color: $color-gray-8;
  border: none;

  /* remove the manual push and instead rely on flex */
  padding-right: 0;

  .sorted-by {
    flex: 1; /* take up remaining space on left */
  }

  .total-documents {
    white-space: nowrap; /* prevent wrapping */
  }
}
