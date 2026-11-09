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


## frontend rtk query
```
// src/features/api/ecommerceApi.ts
import { createApi, fetchBaseQuery } from '@reduxjs/toolkit/query/react';

export const ecommerceApi = createApi({
  reducerPath: 'ecommerceApi',
  baseQuery: fetchBaseQuery({ baseUrl: '/api' }),
  tagTypes: ['Product', 'Review', 'Order'],
  endpoints: (builder) => ({

    // ১. গেট অল প্রোডাক্টস (Infinite Scroll)
    getAllProductsInfinite: builder.query({
      query: ({ limit = 10, cursor, category, search, sortBy, sortOrder }) => {
        const params = new URLSearchParams();
        if (limit) params.append('limit', limit.toString());
        if (cursor) params.append('cursor', cursor);
        if (category) params.append('category', category);
        if (search) params.append('search', search);
        if (sortBy) params.append('sortBy', sortBy);
        if (sortOrder) params.append('sortOrder', sortOrder);

        return `/products?${params.toString()}`;
      },
      serializeQueryArgs: ({ endpointName, queryArgs }) => {
        const { cursor: _cursor, ...rest } = queryArgs;
        return `${endpointName}-${JSON.stringify(rest)}`;
      },
      merge: (currentCache: any, newData: any, { arg }) => {
        if (!arg.cursor) return newData;
        currentCache.data.push(...newData.data);
        currentCache.meta = newData.meta;
      },
      forceRefetch({ currentArg, previousArg }) {
        return currentArg?.cursor !== previousArg?.cursor;
      },
    }),

    // ২. গেট অল রিভিউস (Infinite Scroll)
    getAllReviewsInfinite: builder.query({
      query: ({ limit = 10, cursor, productId, rating, search, sortBy, sortOrder }) => {
        const params = new URLSearchParams();
        if (limit) params.append('limit', limit.toString());
        if (cursor) params.append('cursor', cursor);
        if (productId) params.append('productId', productId);
        if (rating) params.append('rating', rating.toString());
        if (search) params.append('search', search);
        if (sortBy) params.append('sortBy', sortBy);
        if (sortOrder) params.append('sortOrder', sortOrder);

        return `/reviews?${params.toString()}`;
      },
      serializeQueryArgs: ({ endpointName, queryArgs }) => {
        const { cursor: _cursor, ...rest } = queryArgs;
        return `${endpointName}-${JSON.stringify(rest)}`;
      },
      merge: (currentCache: any, newData: any, { arg }) => {
        if (!arg.cursor) return newData;
        currentCache.data.push(...newData.data);
        currentCache.meta = newData.meta;
      },
      forceRefetch({ currentArg, previousArg }) {
        return currentArg?.cursor !== previousArg?.cursor;
      },
    }),

    // ৩. গেট অল অর্ডারস (Infinite Scroll - Admin/Customer Order History)
    getAllOrdersInfinite: builder.query({
      query: ({ limit = 10, cursor, status, userId, search, sortBy, sortOrder }) => {
        const params = new URLSearchParams();
        if (limit) params.append('limit', limit.toString());
        if (cursor) params.append('cursor', cursor);
        if (status) params.append('status', status); // যেমন: pending, processing, delivered, cancelled
        if (userId) params.append('userId', userId);
        if (search) params.append('search', search);
        if (sortBy) params.append('sortBy', sortBy);
        if (sortOrder) params.append('sortOrder', sortOrder);

        return `/orders?${params.toString()}`;
      },
      serializeQueryArgs: ({ endpointName, queryArgs }) => {
        const { cursor: _cursor, ...rest } = queryArgs;
        return `${endpointName}-${JSON.stringify(rest)}`;
      },
      merge: (currentCache: any, newData: any, { arg }) => {
        if (!arg.cursor) return newData;
        currentCache.data.push(...newData.data);
        currentCache.meta = newData.meta;
      },
      forceRefetch({ currentArg, previousArg }) {
        return currentArg?.cursor !== previousArg?.cursor;
      },
    }),

  }),
});

export const {
  useGetAllProductsInfiniteQuery,
  useGetAllReviewsInfiniteQuery,
  useGetAllOrdersInfiniteQuery,
} = ecommerceApi;
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



## Nextjs cursor patination
```
// src/helper/cursorPagination.ts
/* eslint-disable @typescript-eslint/no-explicit-any */
import { Types } from 'mongoose';

export type SortOrder = 'asc' | 'desc';

export interface ICursorPaginationOptions {
  limit?: number | string | null;
  cursor?: string | null;
  sortBy?: string;
  sortOrder?: SortOrder;
  maxLimit?: number;
}

export interface ICursorPaginationResult {
  limit: number;
  fetchLimit: number;
  sortBy: string;
  sortOrder: SortOrder;
  sort: Record<string, 1 | -1>;
  filter: Record<string, unknown>;
}

export interface ICursorPaginationMeta {
  limit: number;
  hasMore: boolean;
  nextCursor: string | null;
  sortBy: string;
  sortOrder: SortOrder;
}

export class InvalidCursorError extends Error {
  constructor() {
    super('Invalid cursor');
    this.name = 'InvalidCursorError';
  }
}

/* ====================== CURSOR ENCODE / DECODE ====================== */
interface CursorPayload {
  v: string | number | boolean | null;
  d: boolean;    
  id: string;    
  s: string;      
  o: SortOrder;   
}

const OBJECT_ID_REGEX = /^[a-f\d]{24}$/i;

const encodeCursor = (payload: CursorPayload): string =>
  Buffer.from(JSON.stringify(payload)).toString('base64url');

const decodeCursor = (cursor: string): CursorPayload => {
  let parsed: any;
  try {
    parsed = JSON.parse(Buffer.from(cursor, 'base64url').toString('utf8'));
  } catch {
    throw new InvalidCursorError();
  }

  const validValue = parsed?.v === null || ['string', 'number', 'boolean'].includes(typeof parsed?.v);

  if (
    !parsed ||
    typeof parsed.id !== 'string' ||
    !OBJECT_ID_REGEX.test(parsed.id) ||
    typeof parsed.d !== 'boolean' ||
    typeof parsed.s !== 'string' ||
    (parsed.o !== 'asc' && parsed.o !== 'desc') ||
    !validValue
  ) {
    throw new InvalidCursorError();
  }
  return parsed as CursorPayload;
};


/* ====================== CURSOR PAGINATION ====================== */
export const calculateCursorPagination = (
  options: ICursorPaginationOptions,
): ICursorPaginationResult => {
  const maxLimit = options.maxLimit ?? 50;
  const rawLimit = Number(options.limit);
  const limit = Number.isFinite(rawLimit) && rawLimit > 0 ? Math.min(Math.floor(rawLimit), maxLimit) : 10;

  const sortBy = options.sortBy || 'createdAt';
  const sortOrder: SortOrder = options.sortOrder || 'desc';
  const dir = sortOrder === 'asc' ? 1 : -1;
  const op = sortOrder === 'asc' ? '$gt' : '$lt';

  const sort: Record<string, 1 | -1> =
    sortBy === '_id' ? { _id: dir } : { [sortBy]: dir, _id: dir };

  let filter: Record<string, unknown> = {};

  if (options.cursor) {
    const c = decodeCursor(options.cursor);
    if (c.s !== sortBy || c.o !== sortOrder) throw new InvalidCursorError();
    const value = c.d ? new Date(c.v as string) : c.v;
    if (c.d && Number.isNaN((value as Date).getTime())) throw new InvalidCursorError();
    const id = new Types.ObjectId(c.id); 
    filter =
      sortBy === '_id'
        ? { _id: { [op]: id } }
        : {
            $or: [
              { [sortBy]: { [op]: value } },
              { [sortBy]: value, _id: { [op]: id } },
            ],
          };
  }
  return { limit, fetchLimit: limit + 1, sortBy, sortOrder, sort, filter };
};


/* ====================== CREATE CURSOR PAGINATION META ====================== */
export const createCursorPaginationMeta = <T extends Record<string, any>>(
  data: T[],
  limit: number,
  sortBy: string = 'createdAt',
  sortOrder: SortOrder = 'desc',
): { data: T[]; meta: ICursorPaginationMeta } => {
  const hasMore = data.length > limit;
  const paginatedData = hasMore ? data.slice(0, limit) : data;

  let nextCursor: string | null = null;
  if (hasMore) {
    const last = paginatedData[paginatedData.length - 1];
    const raw = sortBy === '_id' ? last._id : last[sortBy];

    if (raw === undefined || raw === null) {
      throw new Error(`Cursor sort field "${sortBy}" is null/undefined on a document`);
    }

    const isDate = raw instanceof Date;
    nextCursor = encodeCursor({
      v: isDate
        ? raw.toISOString()
        : typeof raw === 'string' || typeof raw === 'number' || typeof raw === 'boolean'
          ? raw
          : String(raw),
      d: isDate,
      id: String(last._id),
      s: sortBy,
      o: sortOrder,
    });
  }
  return {
    data: paginatedData,
    meta: { limit, hasMore, nextCursor, sortBy, sortOrder },
  };
};

/* ====================== HELPERS ====================== */
export const combineFilters = (
  ...filters: Array<Record<string, unknown> | undefined>
): Record<string, unknown> => {
  const valid = filters.filter(
    (f): f is Record<string, unknown> => !!f && Object.keys(f).length > 0,
  );
  if (valid.length === 0) return {};
  if (valid.length === 1) return valid[0];
  return { $and: valid };
};


export const getCursorParams = (
  searchParams: URLSearchParams,
  config: { allowedSortBy?: string[]; defaultSortBy?: string; defaultSortOrder?: SortOrder } = {},
): ICursorPaginationOptions => {
  const { allowedSortBy = ['createdAt'], defaultSortBy = 'createdAt', defaultSortOrder = 'desc' } = config;
  const sortByParam = searchParams.get('sortBy');
  const sortOrderParam = searchParams.get('sortOrder');

  return {
    limit: searchParams.get('limit'),
    cursor: searchParams.get('cursor'),
    sortBy: sortByParam && allowedSortBy.includes(sortByParam) ? sortByParam : defaultSortBy,
    sortOrder: sortOrderParam === 'asc' || sortOrderParam === 'desc' ? sortOrderParam : defaultSortOrder,
  };
};

```
