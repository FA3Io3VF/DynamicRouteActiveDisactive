# DynamicRouteActiveDisactive
Dynamic FastApi Route Activation/Disactivation

## Basic operation

```python
from fastapi import FastAPI, deprecated

app = FastAPI()

@deprecated(reason="This endpoint is no longer supported.")
@app.get("/old_endpoint")
async def old_endpoint():
    return {"message": "This endpoint is deprecated."}
```

## Solution to dynamic management of route activation

```python
from fastapi import FastAPI, HTTPException
from fastapi.responses import JSONResponse
from sqlalchemy import create_engine, Column, Integer, String, Boolean
from sqlalchemy.orm import sessionmaker, selectinload
from sqlalchemy.ext.declarative import declarative_base
from sqlalchemy.ext.asyncio import create_async_engine, AsyncSession
from typing import List

app = FastAPI()
async_engine = create_async_engine("postgresql+asyncpg://user:password@localhost/dbname", echo=True)
Base = declarative_base()

class Route(Base):
    __tablename__ = "routes"
    id = Column(Integer, primary_key=True, autoincrement=True, index=True)
    name = Column(String(50))
    path = Column(String(50), nullable=False)
    description = Column(String(100), nullable=False)
    active = Column(Boolean, nullable=False, default=True)

    def as_dict(self):
        return {c.name: getattr(self, c.name) for c in self.__table__.columns}

class RouteManager:
    def __init__(self, app):
        self.app = app
        self.routes = []

    async def initialize_cache(self):
        async with async_engine.begin() as conn:
            async with AsyncSession(bind=conn) as session:
                result = await session.execute(selectinload(Route).filter(Route.active == True))
                self.routes = [r.as_dict() for r in result.scalars().all()]

    async def get_active_routes(self):
        if not self.routes:
            await self.initialize_cache()
        return self.routes

    async def disable_route(self, route_name):
        async with async_engine.begin() as conn:
            async with AsyncSession(bind=conn) as session:
                result = await session.execute(selectinload(Route).filter(Route.name == route_name))
                route = result.scalar_one_or_none()
                if not route:
                    raise HTTPException(status_code=404, detail=f"Route '{route_name}' not found")
                route.active = False
                await session.commit()
                self.routes = [r.as_dict() for r in await session.execute(selectinload(Route))]

    async def enable_route(self, route_name):
        async with async_engine.begin() as conn:
            async with AsyncSession(bind=conn) as session:
                result = await session.execute(selectinload(Route).filter(Route.name == route_name))
                route = result.scalar_one_or_none()
                if not route:
                    raise HTTPException(status_code=404, detail=f"Route '{route_name}' not found")
                route.active = True
                await session.commit()
                self.routes = [r.as_dict() for r in await session.execute(selectinload(Route))]

route_manager = RouteManager(app)

@app.on_event("startup")
async def startup_event():
    await route_manager.initialize_cache()

@app.get("/")
async def read_root():
    return {"Hello": "World"}

@app.get("/items")
async def read_item(str = None):
    return {"item_id": str}

@app.get("/routes")
async def get_routes():
    return await route_manager.get_active_routes()

@app.post("/routes")
async def create_route(route: Route):
    async with async_engine.begin() as conn:
        async with AsyncSession(bind=conn) as session:
            session.add(route)
            await session.commit()
            return {"message": "Route created successfully"}

@app.put("/routes/disable/{route_name}")
async def disable_route(route_name: str):
    await route_manager.disable_route(route_name)
    return {"message": f"Route '{route_name}' disabled successfully"}

@app.put("/routes/enable/{route_name}")
async def enable_route(route_name: str):
    await route_manager.enable_route(route_name)
    return {"message": f"Route '{route_name}' enabled successfully"}

@app.put("/routes/{route_id}")
async def update_route(route_id: int, route: Route):
    async with async_engine.begin() as conn:
        async with AsyncSession(bind=conn) as session:
            db_route = await session.get(Route, route_id)
            if not db_route:
                raise HTTPException(status_code=404, detail=f"Route with id {route_id} not found")
            db_route.name = route.name
            db_route.path = route.path
            db_route.description = route.description
            await session.commit()
            return {"message": f"Route {route_id} updated successfully"}

@app.delete("/routes/{route_id}")
async def delete_route(route_id: int):
    async with async_engine.begin() as conn:
        async with AsyncSession(bind=conn) as session:
            db_route = await session.get(Route, route_id)
            if not db_route:
                raise HTTPException(status_code=404, detail=f"Route with id {route_id} not found")
            session.delete(db_route)
            await session.commit()
            return {"message": f"Route {route_id} deleted successfully"}
```
