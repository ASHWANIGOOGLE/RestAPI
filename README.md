from fastapi import FastAPI, HTTPException, Depends, Query
from sqlalchemy import create_engine, Column, Integer, String, Float
from sqlalchemy.orm import sessionmaker, declarative_base
from typing import List

# SQLAlchemy setup
DATABASE_URL = "postgresql://username:password@localhost/dbname"
engine = create_engine(DATABASE_URL)
SessionLocal = sessionmaker(autocommit=False, autoflush=False, bind=engine)
Base = declarative_base()

class Product(Base):
    __tablename__ = "products"
    id = Column(Integer, primary_key=True, index=True)
    name = Column(String, index=True)
    price = Column(Float)
    stock_level = Column(Integer)

Base.metadata.create_all(bind=engine)

# FastAPI setup
app = FastAPI()

# Dependency for database session
def get_db():
    db = SessionLocal()
    try:
        yield db
    finally:
        db.close()

# Pydantic models
class ProductBase(Base):
    price: float
    stock_level: int

class ProductCreate(ProductBase):
    name: str

class ProductUpdate(ProductBase):
    pass

class ProductResponse(ProductBase):
    id: int
    name: str

# Routes
@app.get("/products/{product_id}", response_model=ProductResponse)
def read_product(product_id: int, db: Session = Depends(get_db)):
    product = db.query(Product).filter(Product.id == product_id).first()
    if product is None:
        raise HTTPException(status_code=404, detail="Product not found")
    return product

@app.get("/products", response_model=List[ProductResponse])
def read_products(page: int = Query(1, gt=0), limit: int = Query(10, gt=0), db: Session = Depends(get_db)):
    products = db.query(Product).offset((page - 1) * limit).limit(limit).all()
    return products

@app.patch("/products/{product_id}", response_model=ProductResponse)
def update_product(product_id: int, product_update: ProductUpdate, db: Session = Depends(get_db)):
    product = db.query(Product).filter(Product.id == product_id).first()
    if product is None:
        raise HTTPException(status_code=404, detail="Product not found")

    for key, value in product_update.dict().items():
        setattr(product, key, value)

    db.commit()
    db.refresh(product)
    return product

@app.get("/products/search", response_model=List[ProductResponse])
def search_products(name: str = None, min_price: float = None, max_price: float = None, db: Session = Depends(get_db)):
    query = db.query(Product)

    if name:
        query = query.filter(Product.name.ilike(f"%{name}%"))
    if min_price is not None:
        query = query.filter(Product.price >= min_price)
    if max_price is not None:
        query = query.filter(Product.price <= max_price)

    products = query.all()
    return products
