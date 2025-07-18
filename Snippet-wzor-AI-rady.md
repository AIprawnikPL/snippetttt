
Podsumowanie – kluczowe profesjonalne praktyki
Projektuj API zgodnie ze standardami REST i OpenAPI.
Automatyzuj generowanie SDK i snippetów na podstawie specyfikacji.
Dbaj o bezpieczeństwo, testy, dokumentację i automatyzację wdrożeń.
Zadbaj o wygodę deweloperów: przykłady, playground, gotowe SDK, generator kodu.
Myśl o skalowalności i przyszłym rozwoju (wersjonowanie, modularność).
Chcesz, żebym przygotował szablon projektu (np. strukturę katalogów, pliki startowe) lub gotowy kod dla wybranej części? Jeśli tak – napisz, którą część chcesz zobaczyć jako pierwszą!




Profesjonalny System API dla Kancelarii Prawniczych
1. Architektura i Planowanie
1.1 Domain-Driven Design (DDD)
Zanim zaczniesz kodować, zdefiniuj domenę biznesową:

Domain: Legal Services
Entities: LawFirm, Lawyer, Specialization, Address, Contact
Value Objects: PhoneNumber, Email, TaxNumber
Aggregates: LawFirm (root aggregate)
Services: SearchService, ValidationService
1.2 API Design Principles
RESTful: Używaj standardowych metod HTTP (GET, POST, PUT, DELETE)
Resource-oriented: URL reprezentują zasoby, nie akcje
Stateless: Każde żądanie zawiera wszystkie potrzebne informacje
Versioning: Zawsze wersjonuj API (/api/v1/)
HATEOAS: Linki do powiązanych zasobów
1.3 Standardy JSON API
Struktura odpowiedzi zgodna z JSON:API:

json
{
  "data": {
    "type": "law-firms",
    "id": "123",
    "attributes": {
      "name": "Kowalski & Associates",
      "founded": "2010-01-15"
    },
    "relationships": {
      "lawyers": {
        "data": [
          {"type": "lawyers", "id": "456"}
        ]
      }
    }
  },
  "included": [
    {
      "type": "lawyers",
      "id": "456",
      "attributes": {
        "full_name": "Jan Kowalski",
        "specializations": ["corporate", "tax"]
      }
    }
  ]
}
2. Implementacja Backend API
2.1 Struktura Projektu
law_firm_api/
├── app/
│   ├── __init__.py
│   ├── main.py
│   ├── core/
│   │   ├── config.py
│   │   ├── security.py
│   │   └── exceptions.py
│   ├── models/
│   │   ├── __init__.py
│   │   ├── law_firm.py
│   │   └── lawyer.py
│   ├── schemas/
│   │   ├── __init__.py
│   │   ├── law_firm.py
│   │   └── responses.py
│   ├── api/
│   │   ├── __init__.py
│   │   ├── v1/
│   │   │   ├── __init__.py
│   │   │   ├── endpoints/
│   │   │   │   ├── law_firms.py
│   │   │   │   └── lawyers.py
│   │   │   └── router.py
│   ├── services/
│   │   ├── __init__.py
│   │   ├── law_firm_service.py
│   │   └── search_service.py
│   └── tests/
├── requirements.txt
├── docker-compose.yml
├── Dockerfile
└── README.md
2.2 Modele Danych (models/law_firm.py)
python
from sqlalchemy import Column, String, DateTime, Text, Table, ForeignKey
from sqlalchemy.ext.declarative import declarative_base
from sqlalchemy.orm import relationship
from sqlalchemy.dialects.postgresql import UUID, ARRAY
import uuid
from datetime import datetime
from typing import Optional, List

Base = declarative_base()

# Association table for many-to-many relationship
law_firm_specializations = Table(
    'law_firm_specializations',
    Base.metadata,
    Column('law_firm_id', UUID(as_uuid=True), ForeignKey('law_firms.id')),
    Column('specialization_id', UUID(as_uuid=True), ForeignKey('specializations.id'))
)

class LawFirm(Base):
    __tablename__ = "law_firms"

    id = Column(UUID(as_uuid=True), primary_key=True, default=uuid.uuid4)
    name = Column(String(255), nullable=False, index=True)
    tax_number = Column(String(20), unique=True, nullable=False)
    krs_number = Column(String(20), unique=True, nullable=True)
    founded_date = Column(DateTime, nullable=True)
    description = Column(Text, nullable=True)
    
    # Address (embedded value object)
    street = Column(String(255), nullable=False)
    city = Column(String(100), nullable=False, index=True)
    postal_code = Column(String(10), nullable=False)
    country = Column(String(2), default='PL')
    
    # Contact information
    phone = Column(String(20), nullable=True)
    email = Column(String(255), nullable=True)
    website = Column(String(255), nullable=True)
    
    # Business hours (JSON field)
    business_hours = Column(Text, nullable=True)  # Store as JSON string
    
    # Metadata
    created_at = Column(DateTime, default=datetime.utcnow)
    updated_at = Column(DateTime, default=datetime.utcnow, onupdate=datetime.utcnow)
    is_active = Column(Boolean, default=True)
    
    # Relationships
    lawyers = relationship("Lawyer", back_populates="law_firm")
    specializations = relationship("Specialization", secondary=law_firm_specializations, back_populates="law_firms")

class Lawyer(Base):
    __tablename__ = "lawyers"
    
    id = Column(UUID(as_uuid=True), primary_key=True, default=uuid.uuid4)
    first_name = Column(String(100), nullable=False)
    last_name = Column(String(100), nullable=False, index=True)
    title = Column(String(50), nullable=True)  # dr, prof, etc.
    email = Column(String(255), nullable=True)
    phone = Column(String(20), nullable=True)
    bar_number = Column(String(20), unique=True, nullable=True)  # Numer wpisu na listę adwokatów
    
    law_firm_id = Column(UUID(as_uuid=True), ForeignKey('law_firms.id'))
    law_firm = relationship("LawFirm", back_populates="lawyers")

class Specialization(Base):
    __tablename__ = "specializations"
    
    id = Column(UUID(as_uuid=True), primary_key=True, default=uuid.uuid4)
    name = Column(String(100), nullable=False, unique=True)
    code = Column(String(20), nullable=False, unique=True)  # e.g., 'CIVIL', 'CRIMINAL'
    description = Column(Text, nullable=True)
    
    law_firms = relationship("LawFirm", secondary=law_firm_specializations, back_populates="specializations")
2.3 Pydantic Schemas (schemas/law_firm.py)
python
from pydantic import BaseModel, EmailStr, HttpUrl, Field, validator
from typing import Optional, List, Dict, Any
from datetime import datetime
from uuid import UUID
import re

class AddressSchema(BaseModel):
    street: str = Field(..., min_length=1, max_length=255)
    city: str = Field(..., min_length=1, max_length=100)
    postal_code: str = Field(..., regex=r'^\d{2}-\d{3}$')
    country: str = Field(default='PL', regex=r'^[A-Z]{2}$')

class ContactSchema(BaseModel):
    phone: Optional[str] = Field(None, regex=r'^\+?[1-9]\d{1,14}$')
    email: Optional[EmailStr] = None
    website: Optional[HttpUrl] = None

class SpecializationSchema(BaseModel):
    id: UUID
    name: str
    code: str
    description: Optional[str] = None
    
    class Config:
        from_attributes = True

class LawyerSchema(BaseModel):
    id: UUID
    first_name: str
    last_name: str
    title: Optional[str] = None
    email: Optional[EmailStr] = None
    phone: Optional[str] = None
    bar_number: Optional[str] = None
    
    class Config:
        from_attributes = True

class LawFirmCreateSchema(BaseModel):
    name: str = Field(..., min_length=1, max_length=255)
    tax_number: str = Field(..., regex=r'^\d{10}$')
    krs_number: Optional[str] = Field(None, regex=r'^\d{10}$')
    description: Optional[str] = None
    address: AddressSchema
    contact: ContactSchema
    specialization_ids: List[UUID] = []
    
    @validator('name')
    def validate_name(cls, v):
        if not v.strip():
            raise ValueError('Name cannot be empty')
        return v.strip()

class LawFirmUpdateSchema(BaseModel):
    name: Optional[str] = None
    description: Optional[str] = None
    address: Optional[AddressSchema] = None
    contact: Optional[ContactSchema] = None
    specialization_ids: Optional[List[UUID]] = None

class LawFirmResponseSchema(BaseModel):
    id: UUID
    name: str
    tax_number: str
    krs_number: Optional[str]
    founded_date: Optional[datetime]
    description: Optional[str]
    address: AddressSchema
    contact: ContactSchema
    business_hours: Optional[Dict[str, Any]] = None
    created_at: datetime
    updated_at: datetime
    is_active: bool
    lawyers: List[LawyerSchema] = []
    specializations: List[SpecializationSchema] = []
    
    class Config:
        from_attributes = True

# JSON:API Response Schemas
class JSONAPIResponse(BaseModel):
    data: Any
    included: Optional[List[Any]] = None
    meta: Optional[Dict[str, Any]] = None
    links: Optional[Dict[str, str]] = None

class PaginationMeta(BaseModel):
    total: int
    page: int = 1
    per_page: int = 20
    pages: int
    has_next: bool
    has_prev: bool
2.4 Service Layer (services/law_firm_service.py)
python
from sqlalchemy.orm import Session
from sqlalchemy import and_, or_
from typing import List, Optional, Dict, Any
from uuid import UUID

from ..models.law_firm import LawFirm, Specialization
from ..schemas.law_firm import LawFirmCreateSchema, LawFirmUpdateSchema
from ..core.exceptions import NotFoundError, ValidationError

class LawFirmService:
    def __init__(self, db: Session):
        self.db = db
    
    async def create_law_firm(self, data: LawFirmCreateSchema) -> LawFirm:
        """Create a new law firm with validation."""
        
        # Check if tax number already exists
        existing = self.db.query(LawFirm).filter(LawFirm.tax_number == data.tax_number).first()
        if existing:
            raise ValidationError(f"Law firm with tax number {data.tax_number} already exists")
        
        # Validate specializations
        specializations = []
        if data.specialization_ids:
            specializations = self.db.query(Specialization).filter(
                Specialization.id.in_(data.specialization_ids)
            ).all()
            
            if len(specializations) != len(data.specialization_ids):
                raise ValidationError("Some specialization IDs are invalid")
        
        # Create law firm
        law_firm = LawFirm(
            name=data.name,
            tax_number=data.tax_number,
            krs_number=data.krs_number,
            description=data.description,
            street=data.address.street,
            city=data.address.city,
            postal_code=data.address.postal_code,
            country=data.address.country,
            phone=data.contact.phone,
            email=data.contact.email,
            website=str(data.contact.website) if data.contact.website else None,
            specializations=specializations
        )
        
        self.db.add(law_firm)
        self.db.commit()
        self.db.refresh(law_firm)
        
        return law_firm
    
    async def get_law_firm_by_id(self, law_firm_id: UUID) -> LawFirm:
        """Get law firm by ID with all related data."""
        law_firm = self.db.query(LawFirm).filter(
            and_(LawFirm.id == law_firm_id, LawFirm.is_active == True)
        ).first()
        
        if not law_firm:
            raise NotFoundError(f"Law firm with ID {law_firm_id} not found")
            
        return law_firm
    
    async def search_law_firms(
        self, 
        query: Optional[str] = None,
        city: Optional[str] = None,
        specializations: Optional[List[str]] = None,
        page: int = 1,
        per_page: int = 20
    ) -> tuple[List[LawFirm], int]:
        """Advanced search with filtering and pagination."""
        
        base_query = self.db.query(LawFirm).filter(LawFirm.is_active == True)
        
        # Text search
        if query:
            base_query = base_query.filter(
                or_(
                    LawFirm.name.ilike(f"%{query}%"),
                    LawFirm.description.ilike(f"%{query}%")
                )
            )
        
        # City filter
        if city:
            base_query = base_query.filter(LawFirm.city.ilike(f"%{city}%"))
        
        # Specialization filter
        if specializations:
            base_query = base_query.join(LawFirm.specializations).filter(
                Specialization.code.in_(specializations)
            ).distinct()
        
        # Get total count for pagination
        total = base_query.count()
        
        # Apply pagination
        law_firms = base_query.offset((page - 1) * per_page).limit(per_page).all()
        
        return law_firms, total

class SearchService:
    """Advanced search with Elasticsearch integration (optional)."""
    
    def __init__(self, db: Session, es_client=None):
        self.db = db
        self.es_client = es_client
    
    async def full_text_search(self, query: str, filters: Dict[str, Any] = None) -> List[Dict]:
        """Full-text search with Elasticsearch (if available)."""
        if not self.es_client:
            # Fallback to database search
            return await self._database_search(query, filters)
        
        # Elasticsearch implementation
        search_body = {
            "query": {
                "bool": {
                    "must": [
                        {
                            "multi_match": {
                                "query": query,
                                "fields": ["name^2", "description", "specializations"]
                            }
                        }
                    ]
                }
            }
        }
        
        if filters:
            # Add filters to search body
            pass
        
        # Execute search and return results
        # Implementation depends on your Elasticsearch setup
        pass
2.5 API Endpoints (api/v1/endpoints/law_firms.py)
python
from fastapi import APIRouter, Depends, HTTPException, Query, status
from fastapi.responses import JSONResponse
from sqlalchemy.orm import Session
from typing import List, Optional
from uuid import UUID
import math

from ....database import get_db
from ....services.law_firm_service import LawFirmService
from ....schemas.law_firm import (
    LawFirmCreateSchema, 
    LawFirmUpdateSchema, 
    LawFirmResponseSchema,
    JSONAPIResponse,
    PaginationMeta
)
from ....core.exceptions import NotFoundError, ValidationError
from ....core.security import get_current_user  # If authentication is needed

router = APIRouter(prefix="/law-firms", tags=["Law Firms"])

@router.post(
    "/",
    response_model=JSONAPIResponse,
    status_code=status.HTTP_201_CREATED,
    summary="Create a new law firm",
    description="Creates a new law firm with complete validation and business rules.",
    responses={
        201: {"description": "Law firm created successfully"},
        400: {"description": "Validation error"},
        409: {"description": "Law firm already exists"}
    }
)
async def create_law_firm(
    data: LawFirmCreateSchema,
    db: Session = Depends(get_db),
    # current_user = Depends(get_current_user)  # Uncomment if authentication needed
):
    try:
        service = LawFirmService(db)
        law_firm = await service.create_law_firm(data)
        
        return JSONAPIResponse(
            data={
                "type": "law-firms",
                "id": str(law_firm.id),
                "attributes": LawFirmResponseSchema.from_orm(law_firm).dict()
            },
            meta={"created_at": law_firm.created_at.isoformat()}
        )
    except ValidationError as e:
        raise HTTPException(status_code=400, detail=str(e))

@router.get(
    "/",
    response_model=JSONAPIResponse,
    summary="Search and list law firms",
    description="Advanced search with filtering, sorting, and pagination."
)
async def search_law_firms(
    q: Optional[str] = Query(None, description="Search query for name and description"),
    city: Optional[str] = Query(None, description="Filter by city"),
    specializations: Optional[List[str]] = Query(None, description="Filter by specialization codes"),
    page: int = Query(1, ge=1, description="Page number"),
    per_page: int = Query(20, ge=1, le=100, description="Items per page"),
    sort: Optional[str] = Query("name", description="Sort field"),
    order: Optional[str] = Query("asc", regex="^(asc|desc)$", description="Sort order"),
    db: Session = Depends(get_db)
):
    service = LawFirmService(db)
    law_firms, total = await service.search_law_firms(
        query=q,
        city=city,
        specializations=specializations,
        page=page,
        per_page=per_page
    )
    
    pages = math.ceil(total / per_page)
    
    return JSONAPIResponse(
        data=[
            {
                "type": "law-firms",
                "id": str(firm.id),
                "attributes": LawFirmResponseSchema.from_orm(firm).dict()
            }
            for firm in law_firms
        ],
        meta=PaginationMeta(
            total=total,
            page=page,
            per_page=per_page,
            pages=pages,
            has_next=page < pages,
            has_prev=page > 1
        ).dict(),
        links={
            "self": f"/api/v1/law-firms?page={page}&per_page={per_page}",
            "first": f"/api/v1/law-firms?page=1&per_page={per_page}",
            "last": f"/api/v1/law-firms?page={pages}&per_page={per_page}",
            **({"next": f"/api/v1/law-firms?page={page+1}&per_page={per_page}"} if page < pages else {}),
            **({"prev": f"/api/v1/law-firms?page={page-1}&per_page={per_page}"} if page > 1 else {})
        }
    )

@router.get(
    "/{law_firm_id}",
    response_model=JSONAPIResponse,
    summary="Get law firm by ID",
    responses={
        200: {"description": "Law firm found"},
        404: {"description": "Law firm not found"}
    }
)
async def get_law_firm(
    law_firm_id: UUID,
    db: Session = Depends(get_db)
):
    try:
        service = LawFirmService(db)
        law_firm = await service.get_law_firm_by_id(law_firm_id)
        
        return JSONAPIResponse(
            data={
                "type": "law-firms",
                "id": str(law_firm.id),
                "attributes": LawFirmResponseSchema.from_orm(law_firm).dict(),
                "relationships": {
                    "lawyers": {
                        "data": [
                            {"type": "lawyers", "id": str(lawyer.id)}
                            for lawyer in law_firm.lawyers
                        ]
                    }
                }
            },
            included=[
                {
                    "type": "lawyers",
                    "id": str(lawyer.id),
                    "attributes": {
                        "first_name": lawyer.first_name,
                        "last_name": lawyer.last_name,
                        "title": lawyer.title,
                        "email": lawyer.email
                    }
                }
                for lawyer in law_firm.lawyers
            ]
        )
    except NotFoundError as e:
        raise HTTPException(status_code=404, detail=str(e))
3. Profesjonalny SDK i Generator Kodu
3.1 Python SDK (law_firm_sdk/client.py)
python
import httpx
from typing import List, Optional, Dict, Any, Union
from uuid import UUID
import asyncio
from datetime import datetime
import logging

logger = logging.getLogger(__name__)

class LawFirmAPIError(Exception):
    """Base exception for API errors."""
    def __init__(self, message: str, status_code: int = None, response_data: Dict = None):
        self.message = message
        self.status_code = status_code
        self.response_data = response_data
        super().__init__(message)

class LawFirmAPIClient:
    """Professional client for Law Firm API."""
    
    def __init__(
        self,
        base_url: str = "https://api.lawfirms.example.com",
        api_key: Optional[str] = None,
        timeout: int = 30,
        retry_attempts: int = 3
    ):
        self.base_url = base_url.rstrip('/')
        self.api_key = api_key
        self.timeout = timeout
        self.retry_attempts = retry_attempts
        
        headers = {
            "User-Agent": "LawFirmSDK/1.0.0",
            "Accept": "application/vnd.api+json",
            "Content-Type": "application/vnd.api+json"
        }
        
        if api_key:
            headers["Authorization"] = f"Bearer {api_key}"
        
        self.client = httpx.AsyncClient(
            base_url=self.base_url,
            headers=headers,
            timeout=timeout
        )
    
    async def __aenter__(self):
        return self
    
    async def __aexit__(self, exc_type, exc_val, exc_tb):
        await self.client.aclose()
    
    async def _make_request(
        self,
        method: str,
        endpoint: str,
        params: Optional[Dict] = None,
        json_data: Optional[Dict] = None
    ) -> Dict[str, Any]:
        """Make HTTP request with error handling and retries."""
        
        url = f"/api/v1{endpoint}"
        
        for attempt in range(self.retry_attempts):
            try:
                response = await self.client.request(
                    method=method,
                    url=url,
                    params=params,
                    json=json_data
                )
                
                if response.status_code < 400:
                    return response.json()
                
                # Handle API errors
                error_data = response.json() if response.content else {}
                error_message = error_data.get('detail', f'HTTP {response.status_code}')
                
                raise LawFirmAPIError(
                    message=error_message,
                    status_code=response.status_code,
                    response_data=error_data
                )
                
            except httpx.TimeoutException:
                if attempt == self.retry_attempts - 1:
                    raise LawFirmAPIError("Request timeout")
                await asyncio.sleep(2 ** attempt)  # Exponential backoff
                
            except httpx.NetworkError as e:
                if attempt == self.retry_attempts - 1:
                    raise LawFirmAPIError(f"Network error: {str(e)}")
                await asyncio.sleep(2 ** attempt)
    
    async def create_law_firm(self, law_firm_data: Dict[str, Any]) -> Dict[str, Any]:
        """Create a new law firm."""
        logger.info(f"Creating law firm: {law_firm_data.get('name')}")
        
        return await self._make_request(
            method="POST",
            endpoint="/law-firms",
            json_data=law_firm_data
        )
    
    async def get_law_firm(self, law_firm_id: Union[str, UUID]) -> Dict[str, Any]:
        """Get law firm by ID."""
        return await self._make_request(
            method="GET",
            endpoint=f"/law-firms/{law_firm_id}"
        )
    
    async def search_law_firms(
        self,
        query: Optional[str] = None,
        city: Optional[str] = None,
        specializations: Optional[List[str]] = None,
        page: int = 1,
        per_page: int = 20,
        sort: str = "name",
        order: str = "asc"
    ) -> Dict[str, Any]:
        """Search law firms with advanced filtering."""
        
        params = {
            "page": page,
            "per_page": per_page,
            "sort": sort,
            "order": order
        }
        
        if query:
            params["q"] = query
        if city:
            params["city"] = city
        if specializations:
            params["specializations"] = specializations
        
        return await self._make_request(
            method="GET",
            endpoint="/law-firms",
            params=params
        )
    
    async def update_law_firm(
        self, 
        law_firm_id: Union[str, UUID], 
        update_data: Dict[str, Any]
    ) -> Dict[str, Any]:
        """Update existing law firm."""
        logger.info(f"Updating law firm: {law_firm_id}")
        
        return await self._make_request(
            method="PUT",
            endpoint=f"/law-firms/{law_firm_id}",
            json_data=update_data
        )
    
    async def delete_law_firm(self, law_firm_id: Union[str, UUID]) -> bool:
        """Delete law firm (soft delete)."""
        logger.warning(f"Deleting law firm: {law_firm_id}")
        
        await self._make_request(
            method="DELETE",
            endpoint=f"/law-firms/{law_firm_id}"
        )
        return True

# Convenience wrapper for synchronous usage
class SyncLawFirmAPIClient:
    """Synchronous wrapper for the async client."""
    
    def __init__(self, *args, **kwargs):
        self.async_client = LawFirmAPIClient(*args, **kwargs)
    
    def _run_async(self, coro):
        """Run async function in sync context."""
        try:
            loop = asyncio.get_event_loop()
        except RuntimeError:
            loop = asyncio.new_event_loop()
            asyncio.set_event_loop(loop)
        
        return loop.run_until_complete(coro)
    
    def create_law_firm(self, law_firm_data: Dict[str, Any]) -> Dict[str, Any]:
        return self._run_async(self.async_client.create_law_firm(law_firm_data))
    
    def get_law_firm(self, law_firm_id: Union[str, UUID]) -> Dict[str, Any]:
        return self._run_async(self.async_client.get_law_firm(law_firm_id))
    
    def search_law_firms(self, **kwargs) -> Dict[str, Any]:
        return self._run_async(self.async_client.search_law_firms(**kwargs))
3.2 Zaawansowany Generator Kodu
python
# code_generator/generator.py
from typing import Dict, List, Any, Optional
from dataclasses import dataclass
from enum import Enum
import json
import yaml
from jinja2 import Environment, FileSystemLoader

class Language(Enum):
    PYTHON = "python"
    JAVASCRIPT = "javascript"
    TYPESCRIPT = "typescript"
    JAVA = "java"
    CSHARP = "csharp"
    GO = "go"
    PHP = "php"
    CURL = "curl"

@dataclass
class CodeTemplate:
    language: Language
    template_path: str
    file_extension: str
    dependencies: List[str]

class ProfessionalCodeGenerator:
    """Advanced code generator with template system."""
    
    def __init__(self, templates_dir: str = "templates"):
        self.env = Environment(loader=FileSystemLoader(templates_dir))
        self.templates = {
            Language.PYTHON: CodeTemplate(
                language=Language.PYTHON,
                template_path="python_client.j2",
                file_extension="py",
                dependencies=["httpx", "pydantic"]
            ),
            Language.JAVASCRIPT: CodeTemplate(
                language=Language.JAVASCRIPT,
                template_path="javascript_client.j2",
                file_extension="js",
                dependencies=["axios"]
            ),
            Language.TYPESCRIPT: CodeTemplate(
                language=Language.TYPESCRIPT,
                template_path="typescript_client.j2",
                file_extension="ts",
                dependencies=["axios", "@types/node"]
            )
            # Add more languages...
        }
    
    def generate_client_code(
        self,
        language: Language,
        api_spec: Dict[str, Any],
        config: Optional[Dict[str, Any]] = None
    ) -> str:
        """Generate complete client library code."""
        
        template = self.templates.get(language)
        if not template:
            raise ValueError(f"Template for {language.value} not found")
        
        jinja_template = self.env.get_template(template.template_path)
        
        context = {
            "api_spec": api_spec,
            "config": config or {},
            "dependencies": template.dependencies,
            "language": language.value
        }
        
        return jinja_template.render(**context)
    
    def generate_snippet(
        self,
        language: Language,
        operation: str,
        parameters: Dict[str, Any] = None
    ) -> str:
        """Generate code snippet for specific operation."""
        
        snippets
