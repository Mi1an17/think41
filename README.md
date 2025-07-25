#folder structure

snippet_api/
├── main.py
├── models.py
├── schemas.py
├── database.py
└── crud.py

database.py
python
Copy
Edit
from sqlalchemy import create_engine
from sqlalchemy.ext.declarative import declarative_base
from sqlalchemy.orm import sessionmaker

DATABASE_URL = "sqlite:///./snippets.db"

engine = create_engine(DATABASE_URL, connect_args={"check_same_thread": False})
SessionLocal = sessionmaker(bind=engine, autocommit=False, autoflush=False)
Base = declarative_base()
2. models.py
python
Copy
Edit
from sqlalchemy import Column, String, Text, Integer, ForeignKey, DateTime, func
from sqlalchemy.orm import relationship
from database import Base

class User(Base):
    __tablename__ = "users"
    user_str_id = Column(String, primary_key=True, index=True)
    snippets = relationship("Snippet", back_populates="user")

class Snippet(Base):
    __tablename__ = "snippets"
    id = Column(Integer, primary_key=True, index=True)
    user_str_id = Column(String, ForeignKey("users.user_str_id"))
    snippet_name = Column(String)
    language = Column(String)
    code_content = Column(Text)
    updated_at = Column(DateTime, default=func.now(), onupdate=func.now())

    user = relationship("User", back_populates="snippets")

class SnippetVersion(Base):
    __tablename__ = "snippet_versions"
    id = Column(Integer, primary_key=True, index=True)
    snippet_id = Column(Integer, ForeignKey("snippets.id"))
    code_content = Column(Text)
    created_at = Column(DateTime, default=func.now())
3. schemas.py
python
Copy
Edit
from pydantic import BaseModel
from typing import Optional, List
from datetime import datetime

class SnippetBase(BaseModel):
    snippet_name: str
    language: str
    code_content: str

class SnippetCreate(SnippetBase):
    pass

class SnippetOut(SnippetBase):
    updated_at: datetime

class SnippetListOut(BaseModel):
    snippet_name: str
    language: str
    updated_at: datetime

class SnippetVersionOut(BaseModel):
    id: int
    code_content: str
    created_at: datetime
4. crud.py
python
Copy
Edit
from sqlalchemy.orm import Session
from models import User, Snippet, SnippetVersion
from schemas import SnippetCreate

def get_or_create_user(db: Session, user_str_id: str):
    user = db.query(User).filter(User.user_str_id == user_str_id).first()
    if not user:
        user = User(user_str_id=user_str_id)
        db.add(user)
        db.commit()
        db.refresh(user)
    return user

def create_or_update_snippet(db: Session, user_str_id: str, snippet_data: SnippetCreate):
    user = get_or_create_user(db, user_str_id)
    snippet = db.query(Snippet).filter_by(user_str_id=user_str_id, snippet_name=snippet_data.snippet_name).first()
    if snippet:
        snippet.language = snippet_data.language
        snippet.code_content = snippet_data.code_content
    else:
        snippet = Snippet(user_str_id=user_str_id, **snippet_data.dict())
        db.add(snippet)
    db.commit()
    db.refresh(snippet)

    # Add version
    version = SnippetVersion(snippet_id=snippet.id, code_content=snippet.code_content)
    db.add(version)
    db.commit()
    return snippet

def get_snippet(db: Session, user_str_id: str, name: str):
    return db.query(Snippet).filter_by(user_str_id=user_str_id, snippet_name=name).first()

def list_snippets(db: Session, user_str_id: str):
    return db.query(Snippet).filter_by(user_str_id=user_str_id).all()

def delete_snippet(db: Session, user_str_id: str, name: str):
    snippet = get_snippet(db, user_str_id, name)
    if snippet:
        db.delete(snippet)
        db.commit()
        return True
    return False

def get_versions(db: Session, snippet_id: int):
    return db.query(SnippetVersion).filter_by(snippet_id=snippet_id).all()

def search_snippets_by_keyword(db: Session, user_str_id: str, keyword: str):
    return db.query(Snippet).filter(Snippet.user_str_id == user_str_id,
                                    Snippet.code_content.contains(keyword)).all()
5. main.py
python
Copy
Edit
from fastapi import FastAPI, Depends, HTTPException
from sqlalchemy.orm import Session
from typing import List, Optional

from database import SessionLocal, engine
import models, crud, schemas

models.Base.metadata.create_all(bind=engine)

app = FastAPI()

def get_db():
    db = SessionLocal()
    try:
        yield db
    finally:
        db.close()

@app.post("/users/{user_str_id}/snippets", response_model=schemas.SnippetOut)
def save_snippet(user_str_id: str, snippet: schemas.SnippetCreate, db: Session = Depends(get_db)):
    return crud.create_or_update_snippet(db, user_str_id, snippet)

@app.get("/users/{user_str_id}/snippets/{snippet_name}", response_model=schemas.SnippetOut)
def get_snippet(user_str_id: str, snippet_name: str, db: Session = Depends(get_db)):
    snippet = crud.get_snippet(db, user_str_id, snippet_name)
    if not snippet:
        raise HTTPException(status_code=404, detail="Snippet not found")
    return snippet

@app.get("/users/{user_str_id}/snippets", response_model=List[schemas.SnippetListOut])
def list_user_snippets(user_str_id: str, db: Session = Depends(get_db)):
    return crud.list_snippets(db, user_str_id)

@app.delete("/users/{user_str_id}/snippets/{snippet_name}", status_code=204)
def delete(user_str_id: str, snippet_name: str, db: Session = Depends(get_db)):
    if not crud.delete_snippet(db, user_str_id, snippet_name):
        raise HTTPException(status_code=404, detail="Snippet not found")

@app.get("/users/{user_str_id}/snippets/{snippet_name}/versions", response_model=List[schemas.SnippetVersionOut])
def list_versions(user_str_id: str, snippet_name: str, db: Session = Depends(get_db)):
    snippet = crud.get_snippet(db, user_str_id, snippet_name)
    if not snippet:
        raise HTTPException(status_code=404, detail="Snippet not found")
    return crud.get_versions(db, snippet.id)

@app.get("/users/{user_str_id}/snippets/search", response_model=List[schemas.SnippetListOut])
def search_snippets(user_str_id: str, keyword: str, db: Session = Depends(get_db)):
  return crud.search_snippets_by_keyword(db, user_str_id, keyword)
