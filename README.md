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

## backend code

```

/* ================= CURSOR PAGINATION INTERFACE ================= */
export interface ICursorPaginationOptions {
    limit?: number;
    cursor?: string | Date;
    sortBy?: string;
    sortOrder?: "asc" | "desc";
    search?: string;
    searchFields?: string[];
};


/* ================= CURSOR PAGINATION RESULT INTERFACE ================= */
export interface ICursorPaginationResult {
    limit: number;
    sortBy: string;
    sortOrder: "asc" | "desc";
    filter: any;
};


/* ================= CALCULATE CURSOR PAGINATION ================= */
export const calculateCursorPagination = (options: ICursorPaginationOptions): ICursorPaginationResult => {
    // Default values set if not provided
    const limit = options.limit && options.limit > 0 ? options.limit : 10;
    const sortBy = options.sortBy || 'createdAt';
    const sortOrder = options.sortOrder || 'desc';
    
    // Filter object initialize
    let filter: any = {};

    // Search filtering logic
    if (options.search && options.search.trim() !== '') {
        const searchTerm = options.search.trim();
        const searchFields = options.searchFields || ['name', 'title', 'description'];
        
        // MongoDB $or operator for multiple fields search
        filter.$or = searchFields.map(field => ({
            [field]: {
                $regex: searchTerm,     
                $options: 'i'           
            }
        }));
    }
    
    // Cursor filtering
    if (options.cursor) {
        const cursorDate = typeof options.cursor === 'string' 
            ? new Date(options.cursor) 
            : options.cursor;
        
        if (sortOrder === 'desc') {
            filter[sortBy] = { $lt: cursorDate };
        } else {
            filter[sortBy] = { $gt: cursorDate };
        }
    }

    return {
        limit,
        sortBy,
        sortOrder,
        filter,
    };
};


/* ================= CURSOR PAGINATION INTERFACE ================= */
export interface ICursorPaginationMeta<T> {
    limit: number;
    hasMore: boolean;
    nextCursor: Date | string | null;
    sortBy: string;
    sortOrder: "asc" | "desc";
    totalFetched?: number;            
    search?: string;                   
};


/* ================= CREATE CURSOR PAGINATION META  ================= */
export const createCursorPaginationMeta = <T extends Record<string, any>>(
    data: T[],
    limit: number,
    sortBy: string = 'createdAt',
    sortOrder: "asc" | "desc" = 'desc',
    search?: string                   
): { data: T[]; meta: ICursorPaginationMeta<T> } => {
    const hasMore = data.length > limit;
    
    // Extra item remove 
    const paginatedData = hasMore ? data.slice(0, -1) : data;
    
    const lastItem = paginatedData[paginatedData.length - 1];
    const nextCursor = hasMore && lastItem?.[sortBy] ? lastItem[sortBy] : null;
    
    return {
        data: paginatedData,
        meta: {
            limit,
            hasMore,
            nextCursor,
            sortBy,
            sortOrder,
            totalFetched: paginatedData.length,  
            ...(search && { search }),           
        },
    };
};

```
