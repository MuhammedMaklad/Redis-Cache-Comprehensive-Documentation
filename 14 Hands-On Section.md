# 14. Hands-On Section

Theory is good, but practical experience is invaluable. This section will guide you through setting up Redis locally and integrating it into a sample project.

### Step-by-Step Redis Setup

### 1. Local Installation (macOS/Linux)

**macOS (using Homebrew)**:

```bash
brew install redis
brew services start redis # Start Redis as a background service
redis-cli ping # Should return PONG
```

**Linux (Ubuntu/Debian)**:

```bash
sudo apt update
sudo apt install redis-server
sudo systemctl enable redis-server # Enable Redis to start on boot
sudo systemctl start redis-server # Start Redis service
redis-cli ping # Should return PONG
```

### 2. Docker Setup

Using Docker is the recommended way for local development as it provides an isolated and reproducible environment.

1. **Pull the Redis Docker image**:
    
    ```bash
    docker pull redis
    ```
    
2. **Run a Redis container**:
    
    ```bash
    docker run --name my-redis -p 6379:6379 -d redis redis-server --appendonly yes
    ```
    
    - `-name my-redis`: Assigns a name to your container.
    - `p 6379:6379`: Maps port 6379 on your host to port 6379 in the container.
    - `d`: Runs the container in detached mode (background).
    - `redis-server --appendonly yes`: Starts Redis with AOF persistence enabled.
3. **Connect using `redis-cli` from your host**:
    
    ```bash
    redis-cli ping # Should return PONG
    ```
    
    Or connect from inside the container:
    
    ```bash
    docker exec -it my-redis redis-cli
    ```
    

### Sample Project (Backend + Redis)

Let's create a simple Node.js API that uses Redis for caching.

### Project Setup

1. **Create a new project directory**:
    
    ```bash
    mkdir redis-cache-api
    cd redis-cache-api
    ```
    

2. Initialize Node.js project and install dependencies:    

```bash
npm init -y
npm install express ioredis axios
```

3.  **Create `index.js`**:

```jsx
import express, { Request, Response, NextFunction } from "express";
import Redis from "ioredis";
import axios from "axios"; // Although imported, it wasn't used in the original logic, but kept for consistency.

// Initialize Express app
const app = express();
const port: number = 3000;

// Connect to Redis
// Note: ioredis types are included in the package or via @types/ioredis if using older versions
const redis = new Redis({
  port: 6379,
  host: "127.0.0.1", 
});

redis.on("connect", () => console.log("Connected to Redis!"));
redis.on("error", (err: Error) => console.error("Redis Client Error", err));

// Define an interface for the Product structure
interface Product {
  id: string | number;
  name: string;
  description: string;
  price: number;
  source: string;
}

// Simulate a slow external API call
const fetchProductFromExternalAPI = async (productId: string): Promise<Product> => {
  console.log(`Fetching product ${productId} from external API...`);
  
  // Simulate network delay
  await new Promise((resolve) => setTimeout(resolve, 2000));
  
  return {
    id: productId,
    name: `Product ${productId}`,
    description: `This is product number ${productId}.`,
    price: Math.floor(Math.random() * 100) + 10,
    source: "External API",
  };
};

// API endpoint to get product details with caching
app.get("/products/:id", async (req: Request, res: Response): Promise<void> => {
  const productId: string = req.params.id;
  const cacheKey: string = `product:${productId}`;

  try {
    // 1. Check cache
    const cachedProduct: string | null = await redis.get(cacheKey);
    
    if (cachedProduct) {
      console.log(`Cache hit for product ${productId}`);
      // Parse the JSON string back to an object before sending
      return res.json(JSON.parse(cachedProduct));
    }

    // 2. Cache miss, fetch from external API
    console.log(`Cache miss for product ${productId}`);
    const product: Product = await fetchProductFromExternalAPI(productId);

    // 3. Store in cache with TTL (e.g., 60 seconds)
    // redis.setex returns a promise that resolves to "OK"
    await redis.setex(cacheKey, 60, JSON.stringify(product));
    console.log(`Stored product ${productId} in cache.`);

    res.json(product);
  } catch (error: unknown) {
    console.error("Error fetching product:", error);
    
    // Type guard to handle error message safely
    const errorMessage = error instanceof Error ? error.message : "An unknown error occurred";
    
    res.status(500).json({ error: "Failed to fetch product", details: errorMessage });
  }
});

app.listen(port, () => {
  console.log(`API listening at http://localhost:${port}`);
});
```

### Testing Redis Usage

1. **Start the Node.js API**:
    
    ```bash
    node index.js
    ```
    
2. **Make requests to the API**:
    - Open your browser or use `curl`:
        
        ```bash
        curl <http://localhost:3000/products/1>
        ```
        
    - The first request will take ~2 seconds (simulated API delay) and you'll see "Cache miss" in the API logs.
    - Subsequent requests (within 60 seconds) will be almost instant, and you'll see "Cache hit" in the API logs.
    - After 60 seconds, the cache entry will expire, and the next request will again be a cache miss.

This hands-on example demonstrates the Cache-Aside pattern, showing how Redis can significantly improve the performance of a data-intensive application.

---

## 🚀 Final Goal

By the end of this documentation, the reader should:

- Understand Redis deeply (not just use it)
- Be able to design systems using Redis
- Know trade-offs and production considerations
- Think like a senior engineer when using Redis