# cursor_paginaiton_setup


## backend standard response

```
{
  "success": true,
  "data": [...], 
  "meta": {
    "nextCursor": "65f1...", // পরবর্তী পেজের জন্য শেষ আইটেমের ID বা Timestamp
    "hasMore": true,         // আরও ডাটা আছে কিনা
    "totalCount": 100        // ঐ ক্যাটাগরিতে মোট ডাটা সংখ্যা
  }
}
```

## frontend standard response

```
const [page, setPage] = useState(1);
const [cursors, setCursors] = useState<(string | undefined)[]>([undefined]);

const { data, isFetching } = useGetReviewsQuery({
  limit: 10,
  cursor: cursors[page - 1], // বর্তমান পেজের জন্য কার্সর
  propertyId: "123"          // অন্যান্য ফিল্টার
});

const handlePageChange = (targetPage: number) => {
  setPage(targetPage);
  
  // যদি আমরা পরবর্তী নতুন পেজে যাই এবং 'nextCursor' থাকে
  const nextCursor = data?.meta?.nextCursor;
  if (nextCursor && targetPage === cursors.length) {
    setCursors((prev) => [...prev, nextCursor]);
  }
};
```