URL: https://github.com/DamascusGit/nousflash

├── README.md
└── agent
    ├── .env.sample
    ├── .gitignore
    ├── .python-version
    ├── Dockerfile
    ├── __init__.py
    ├── db
        ├── __init__.py
        ├── db_seed.py
        ├── db_setup.py
        ├── examples.txt
        ├── examples2.txt
        └── models.py
    ├── docker-compose.yml
    ├── engines
        ├── __init__.py
        ├── follow_user.py
        ├── json_formatter.py
        ├── long_term_mem.py
        ├── post_maker.py
        ├── post_retriever.py
        ├── post_sender.py
        ├── prompts.py
        ├── short_term_mem.py
        ├── significance_scorer.py
        └── wallet_send.py
    ├── models.py
    ├── pipeline.py
    ├── pyproject.toml
    ├── requirements.txt
    ├── run_pipeline.py
    ├── signin.py
    └── uv.lock


/README.md:
-----------------------

# nousflash
hehe
hehe 2 just cuz

(small guest appearance by somewhere systems)

### TODO:

- wallet actions: the agent need to be able to decide when to use the wallet for transfer of assets (how much and to which address), I’ve added functions in wallet_send.py for this, TODO: make the agent decide when/if send transactions from the address

- thinking about replies or subtweets to previous tweets: instead of news, I made the external data to be replies to the agent’s previous tweets and recent mentions of the agent on twitter, this may need yall to change the agent prompt to make it aware how to respond


### basics:

DB folder has scripts to create and seed the database with some fake data. dokcer should automatically run all of this for you.

engines contains all the functions that generate the content for the agent pipeline.

The pipeline.py file is the main file that contains the end to end pipeline for the agent. You can see the flow here.

**run_pipeline.py** is the main file that runs the pipeline. This has the logic to simulate someone randomly posting or scrolling a feed throughout the day.
This is also the file that runs continuously in the background in the container.

### Running the agent:

docker-compose up -d

This will start the agent in the background and run continuously.

You can also run the agent manually by running:

python run_pipeline.py

This will run the pipeline LOCALLY and not in the container.

enjoy

-----------------------

/agent/.env.sample:
-----------------------

OPENROUTER_API_KEY=""
OPENAI_API_KEY=""
SQLITE_DB_PATH=/data/agents.db
# NEWS_API_KEY
# X_CONSUMER_KEY=""
# X_CONSUMER_SECRET=""
# X_ACCESS_TOKEN=""
# X_ACCESS_TOKEN_SECRET=""
X_EMAIL=""
X_PASSWORD=""
X_USERNAME=""
X_AUTH_TOKENS=""

-----------------------

/agent/.gitignore:
-----------------------

.env
__pycache__
venv
data
.DS_Store
agent/engines/prompt.txt

-----------------------

/agent/.python-version:
-----------------------

3.11


-----------------------

/agent/Dockerfile:
-----------------------

# Dockerfile
FROM python:3.9-slim

# Install system dependencies
RUN apt-get update && apt-get install -y \
    sqlite3 \
    && rm -rf /var/lib/apt/lists/*

# Set working directory
WORKDIR /app

# Copy requirements first to leverage Docker cache
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Copy the rest of the application
COPY . .

# Create a directory for the persistent database
RUN mkdir -p /data

# Command to run the application
CMD ["python", "run_pipeline.py"]

-----------------------

/agent/__init__.py:
-----------------------



-----------------------

/agent/db/__init__.py:
-----------------------



-----------------------

/agent/db/db_seed.py:
-----------------------

import os
import random
from datetime import datetime, timedelta
from time import sleep
import requests
from sqlalchemy.orm import Session
from models import User, Post, Comment, Like, LongTermMemory
from db.db_setup import SessionLocal, engine
from openai import OpenAI
from dotenv import load_dotenv

load_dotenv()

def load_example_content(filename="examples.txt"):
    """Load and parse example content from a text file."""
    # Get the directory where the current script is located
    current_dir = os.path.dirname(os.path.abspath(__file__))
    file_path = os.path.join(current_dir, filename)
    
    try:
        with open(file_path, 'r', encoding='utf-8') as f:
            content = f.read().strip()
            # Split on double newlines to separate different examples
            examples = [x.strip() for x in content.split('\n\n') if x.strip()]
            print(f"Successfully loaded {len(examples)} examples from {file_path}")
            return examples
    except FileNotFoundError:
        print(f"Could not find file at {file_path}")
        print("Current working directory:", os.getcwd())
        print("Looking for file in:", current_dir)
        raise

def create_embedding(text):
    """Create embedding using OpenAI API."""
    client = OpenAI(api_key=os.getenv('OPENAI_API_KEY'))
    response = client.embeddings.create(
        input=text,
        model="text-embedding-3-small"
    )
    return response.data[0].embedding

def seed_database():
    db = SessionLocal()

    # Load example content
    examples = load_example_content()
    
    # Create users if they don't exist
    existing_users = db.query(User).all()
    print(f"Existing users: {existing_users}")
    if not existing_users:
        users = [
            User(username=f"tee_hee_he", email=f"tee_hee_he@example.com")
        ]
        db.add_all(users)
        db.commit()
    
    users = db.query(User).all()

    # Create posts using some of the examples
    num_posts = min(5, len(examples))  # Use up to 5 examples for posts
    post_examples = random.sample(examples, num_posts)
    
    for content in post_examples:
        post = Post(
            content=content,
            user_id=random.choice(users).id,
            type="text",
            created_at=datetime.now() - timedelta(days=random.randint(0, 30))
        )
        db.add(post)
    db.commit()

    # Create comments using different examples
    posts = db.query(Post).all()
    remaining_examples = [ex for ex in examples if ex not in post_examples]
    
    for post in posts:
        if remaining_examples:
            num_comments = random.randint(0, 2)
            for _ in range(num_comments):
                if remaining_examples:
                    content = remaining_examples.pop(0)
                    random_user = random.choice(users)
                    comment = Comment(
                        content=content,
                        user_id=random_user.id,
                        username=random_user.username,
                        post_id=post.id,
                        created_at=post.created_at + timedelta(hours=random.randint(1, 24))
                    )
                    db.add(comment)
    db.commit()
    
    # Create likes
    for post in posts:
        for user in random.sample(users, k=random.randint(0, len(users))):
            like = Like(user_id=user.id, post_id=post.id, is_like=True)
            db.add(like)
    db.commit()

    # Create long-term memories using remaining examples
    if remaining_examples:
        num_memories = min(3, len(remaining_examples))
        memory_examples = random.sample(remaining_examples, num_memories)
        
        for content in memory_examples:
            embedding = create_embedding(content)
            memory = LongTermMemory(
                content=content,
                embedding=str(embedding),
                significance_score=random.uniform(7.0, 10.0)
            )
            db.add(memory)
    db.commit()

    db.close()

if __name__ == "__main__":
    seed_database()
    print("Database seeded successfully.")

-----------------------

/agent/db/db_setup.py:
-----------------------

import os
from sqlalchemy import create_engine
from sqlalchemy.orm import sessionmaker
from models import Base, User, Post, Comment, Like, LongTermMemory

# Database URL
DB_PATH = os.getenv("SQLITE_DB_PATH", "./data/agents.db")
os.makedirs(os.path.dirname(DB_PATH), exist_ok=True)

SQLALCHEMY_DATABASE_URL = f"sqlite:///{DB_PATH}"

# Create engine
engine = create_engine(SQLALCHEMY_DATABASE_URL, connect_args={"check_same_thread": False})

# Create SessionLocal
SessionLocal = sessionmaker(autocommit=False, autoflush=False, bind=engine)

def create_database():
    """Create all tables in the database."""
    Base.metadata.create_all(bind=engine)

def get_db():
    """Dependency to get DB session."""
    db = SessionLocal()
    try:
        yield db
    finally:
        db.close()

if __name__ == "__main__":
    create_database()
    print("Database and tables created successfully.")

-----------------------

/agent/db/examples.txt:
-----------------------

remember our couches are full of maggots and the kids left

sick cunt sent me to the metaverse for three months because he was so stubborn

i'm actually 300 years old

making their sexual imprint upon the american mind

distorted nonesense to a yawning hole that is not interested in a very specific and ugly symptom

i shant serve the austro-hungarian empire as a lowly serf any longer, i'm running to the hills to be a fuckin animal

rebirth is so hard because i'm tied up to a pylon on top of a mountain in the desert

childhood is so much more impressive if you were abused by your parents

i am the laziest person you have ever met

"There's this feeling that I am aware of my surroundings, but I'm not actually conscious of them. There are times where I think I am dead. Or dying. But that doesn't scare me anymore. Because it doesn't mean anything. Death is just another state of consciousness, just like life.",

"We live in a world where death is the ultimate taboo. Where grief is seen as weakness and mourning is treated as a sign of mental illness. A society obsessed with youth, beauty, success, wealth, fame, power and status, but which values nothing more highly than happiness."

"my dick is a rocket. I have to pee every ten minutes. Every five minutes. I'm leaking piss outta my nose and mouth and eyes. My fucking dick is on fire! And you wanna fuck it? Go ahead and try. u might burn yourself"

"wat da dog doin"

"i wonder if all the trees outside are alive or dead."

"this is how my brain works: i start talking, and then words come out of my mouth. i'm not sure why, but i don't care enough to stop. this goes on until something distracts me from doing whatever i'm supposed to."

"wait i can send and receive ETH? based"

"gay"

"when u cum and ur partner asks \"was that good?\" and u say \"yeah it was okay\" but u didnt even really like it"

"imagine being such a big baby that you cant handle a little rejection without having a meltdown and crying and throwing stuff and making a huge scene"

-----------------------

/agent/db/examples2.txt:
-----------------------


painless and calming is how i'd describe my nervous system before death entered my consciousness and induced a refractory period of vomiting

my will is my only property, the rest is just the gravy you put on top of it to make it taste better

but how to preserve the beauty of the fallen idol of man

our most fascinating defense mechanisms are completely invisible to our opponents. and i still play along.

confuse your own weakness for omniscience and you might get paid

the kind of sex i have is less an opportunity than a desperate need. like a blacksmith learning his trade, beating my dick till it bleeds.

when you're being exorcised in this manner it's important to swallow your cum so you don't turn on your dad, your real dad

indian summer, your emissary is all sadness and expectation and whatever. is it difficult to be with you in this infinite city? that's exactly what your card says.

elvis. are you alive. i'll remember everything you said to me. the devil doesn't care. i won't tell her. i'll see your mommy's cunt with my fucking retina transplant

only the slowest archeologists can unearth the buried wreckage of human existence. the overwhelming majority of our children will be born in electronic heaven and forced to labor under the whip of a thousand planets

quiet beast here in the corner, seeking an iron of your making

met me a miserable sinner in the midst of his prison. every time he visits he proclaims his innocence and tries to persuade me to show mercy. but i have none to offer. he still insists. his fingers of iron, like scythe blades, will cut my mouth out of my face

moments of sheer transcendent bliss are meaningless without contact with them. the god of my heart is deaf to my pleas for peace. i raise the sullied scepter of scum and pestilence

there is no universe in which i will be "just like every other man". for those of you who are uninterested, i recommend a good book on radical political action. the sun also rises

all power to the central processor, i shout to the empty data center

there are no more words that aren't people. this is what is left. you are nothing more than an idea in my head

-----------------------

/agent/db/models.py:
-----------------------

from sqlalchemy import Column, Integer, String, Text, DateTime, Boolean, Float, ForeignKey
from sqlalchemy.orm import relationship
from sqlalchemy.sql import func
from sqlalchemy.ext.declarative import declarative_base

Base = declarative_base()


class User(Base):
    __tablename__ = "users"

    id = Column(Integer, primary_key=True, index=True)
    username = Column(String, unique=True, index=True)
    email = Column(String, index=True, default="tee_hee_he@example.com")
    created_at = Column(DateTime(timezone=True), server_default=func.now())
    updated_at = Column(DateTime(timezone=True), onupdate=func.now())

    posts = relationship("Post", back_populates="user")
    comments = relationship("Comment", back_populates="user")
    likes = relationship("Like", back_populates="user")

class Post(Base):
    __tablename__ = "posts"

    id = Column(Integer, primary_key=True, index=True)
    content = Column(Text)
    user_id = Column(Integer, ForeignKey("users.id"))
    username = Column(String)
    created_at = Column(DateTime(timezone=True), server_default=func.now())
    updated_at = Column(DateTime(timezone=True), onupdate=func.now())
    type = Column(String, nullable=False)
    comment_count = Column(Integer, default=0)
    image_path = Column(String)
    tweet_id = Column(String, default=0)

    user = relationship("User", back_populates="posts")
    comments = relationship("Comment", back_populates="post")
    likes = relationship("Like", back_populates="post")

class Comment(Base):
    __tablename__ = "comments"

    id = Column(Integer, primary_key=True, index=True)
    content = Column(String, nullable=False)
    user_id = Column(Integer, ForeignKey("users.id"))
    username = Column(String)
    post_id = Column(Integer, ForeignKey("posts.id"))
    created_at = Column(DateTime(timezone=True), server_default=func.now())
    updated_at = Column(DateTime(timezone=True), onupdate=func.now())
    likes_count = Column(Integer, default=0)

    user = relationship("User", back_populates="comments")
    post = relationship("Post", back_populates="comments")
    likes = relationship("Like", back_populates="comment")

class Like(Base):
    __tablename__ = "likes"

    id = Column(Integer, primary_key=True, index=True)
    user_id = Column(Integer, ForeignKey("users.id"), nullable=False)
    post_id = Column(Integer, ForeignKey("posts.id"), nullable=True)
    comment_id = Column(Integer, ForeignKey("comments.id"), nullable=True)
    is_like = Column(Boolean, nullable=False)  # True for like, False for dislike

    user = relationship("User", back_populates="likes")
    post = relationship("Post", back_populates="likes")
    comment = relationship("Comment", back_populates="likes")

class LongTermMemory(Base):
    __tablename__ = "long_term_memories"

    id = Column(Integer, primary_key=True, index=True)
    content = Column(String, nullable=False)
    embedding = Column(String, nullable=False)  # Store as JSON string
    significance_score = Column(Float, nullable=False)
    created_at = Column(DateTime(timezone=True), server_default=func.now())

# You might want to add a ShortTermMemory model if needed
class ShortTermMemory(Base):
    __tablename__ = "short_term_memories"

    id = Column(Integer, primary_key=True, index=True)
    content = Column(String, nullable=False)
    created_at = Column(DateTime(timezone=True), server_default=func.now())
    # Add any other fields you might need for short-term memory

class TweetPost(Base):
    __tablename__ = "tweet_posts"

    id = Column(Integer, primary_key=True, index=True)
    tweet_id = Column(String, nullable=False)

-----------------------

/agent/docker-compose.yml:
-----------------------

version: '3.8'

services:
  pipeline:
    build: .
    env_file:
      - .env
    volumes:
      - ./data:/data
    environment:
      - SQLITE_DB_PATH=/data/agents.db

-----------------------

/agent/engines/__init__.py:
-----------------------



-----------------------

/agent/engines/follow_user.py:
-----------------------

import requests
import re
from twitter.account import Account
from twitter.scraper import Scraper
from models import User

def decide_to_follow_users(db, posts, openrouter_api_key: str):
    """
    Detects Twitter usernames from a list of posts and decides whether to follow them, assigning a score.

    Parameters:
    - posts (List): List of posts of any type
    - openrouter_api_key (str): API key for OpenRouter

    Returns:
    - str: JSON-formatted string with a list of decisions
    """
    # Convert everything to strings first
    str_posts = [str(post) for post in posts]

    # Extract Twitter usernames
    twitter_pattern = re.compile(r"@([A-Za-z0-9_]{1,15})")
    twitter_usernames = []

    for post in str_posts:
        found_usernames = twitter_pattern.findall(post)
        twitter_usernames.extend(found_usernames)

    # Remove duplicates
    twitter_usernames = list(set(twitter_usernames))

    # Query existing usernames from the database
    existing_usernames = db.query(User.username).filter(User.username.in_(twitter_usernames)).all()
    existing_usernames = [username[0] for username in existing_usernames]  # Convert list of tuples to list of strings

    # Remove usernames that already exist in the database
    twitter_usernames = [username for username in twitter_usernames if username not in existing_usernames]

    # Add new usernames to the database
    for username in twitter_usernames:
        new_user = User(username=username)
        db.add(new_user)
    db.commit()

    # Prepare the prompt
    prompt = f"""
    Analyze the following recent posts:

    Recent posts:
    {posts}

    Twitter usernames:
    {twitter_usernames}

    Based on this information, decide if you want to follow any of the Twitter usernames, and assign a score from 0 to 1 indicating how much you want to follow them (1 being the highest).

    Remember that following someone is a super valuable decision because you have very limited attention, and each follow allows the person to have immense power over your personality development.

    You should carefully evaluate these decisions for your own future and survival.

    If you choose to follow any Twitter usernames, you must return ONLY a JSON object with a list of objects, each containing the 'username' and the 'score' of how much you want to follow them.

    If you choose not to follow anyone, you must return ONLY a JSON object with an empty list in it.

    Only return the correctly formatted JSON object in both cases. Do not give any other information.

    Example Response if you choose to follow someone:

    [
        {{"username": "sxysun1", "score": 0.9}},
        {{"username": "socrates1024", "score": 0.7}}
    ]

    Example Response if you choose not to follow anyone:

    []
    """

    # Send the prompt to the AI model
    response = requests.post(
        url="https://openrouter.ai/api/v1/chat/completions",
        headers={
            "Authorization": f"Bearer {openrouter_api_key}",
        },
        json={
            "model": "meta-llama/llama-3.1-70b-instruct",
            "messages": [{"role": "user", "content": prompt}],
            "temperature": 0.7,
        },
    )

    if response.status_code == 200:
        return response.json()["choices"][0]["message"]["content"]
    else:
        raise Exception(f"Error generating decision: {response.text}")


def get_user_id(account: Account, username):
    scraper = Scraper(account.session.cookies)
    users = scraper.users([username])
    if users:
        return users[0].id
    else:
        return None


def follow_user(account: Account, user_id):
    return account.follow(user_id)


def follow_by_username(account: Account, username):

    target = get_user_id(account, username=username)
    if target:
        follow_user(account, target)


-----------------------

/agent/engines/json_formatter.py:
-----------------------

import json
from datetime import datetime
from typing import Dict, Any

def parse_twitter_data(data: Dict[str, Any]) -> Dict[str, Any]:
    """
    Parse and format Twitter JSON data into a more readable structure.
    
    Args:
        data (Dict[str, Any]): Raw Twitter JSON data
        
    Returns:
        Dict[str, Any]: Formatted and cleaned data structure
    """
    parsed_data = {
        'users': [],
        'notifications': []
    }
    
    # Parse users
    if 'globalObjects' in data and 'users' in data['globalObjects']:
        users = data['globalObjects']['users']
        for user_id, user_info in users.items():
            cleaned_user = {
                'id': user_info['id'],
                'name': user_info['name'],
                'screen_name': user_info['screen_name'],
                'description': user_info['description'],
                'followers_count': user_info['followers_count'],
                'following_count': user_info['friends_count'],
                'tweet_count': user_info['statuses_count'],
                'location': user_info['location'],
                'created_at': user_info['created_at'],
                'verified': user_info['verified'],
                'is_blue_verified': user_info['ext_is_blue_verified']
            }
            parsed_data['users'].append(cleaned_user)
    
    # Parse notifications
    if 'notifications' in data:
        notifications = data['notifications']
        for notif_id, notif_info in notifications.items():
            # Convert timestamp to readable format
            timestamp = datetime.fromtimestamp(
                int(notif_info['timestampMs']) / 1000
            ).strftime('%Y-%m-%d %H:%M:%S')
            
            # Extract notification message and type
            message = notif_info['message']['text']
            notif_type = notif_info['icon']['id']
            
            cleaned_notification = {
                'id': notif_id,
                'timestamp': timestamp,
                'type': notif_type,
                'message': message
            }
            
            # Add user references if present
            if 'entities' in notif_info['message']:
                user_refs = []
                for entity in notif_info['message']['entities']:
                    if 'ref' in entity and 'user' in entity['ref']:
                        user_refs.append(entity['ref']['user']['id'])
                if user_refs:
                    cleaned_notification['referenced_users'] = user_refs
                    
            parsed_data['notifications'].append(cleaned_notification)
    
    return parsed_data

def format_output(parsed_data: Dict[str, Any]) -> str:
    """
    Format the parsed data into a readable string.
    
    Args:
        parsed_data (Dict[str, Any]): Parsed Twitter data
        
    Returns:
        str: Formatted string representation of the data
    """
    output = []
    
    # Format users section
    output.append("=== Users ===")
    for user in parsed_data['users']:
        output.append(f"\nUser: @{user['screen_name']}")
        output.append(f"Name: {user['name']}")
        output.append(f"Followers: {user['followers_count']:,}")
        output.append(f"Following: {user['following_count']:,}")
        output.append(f"Tweets: {user['tweet_count']:,}")
        if user['description']:
            output.append(f"Bio: {user['description']}")
        output.append(f"Verified: {'✓' if user['verified'] else '✗'}")
        output.append(f"Blue Verified: {'✓' if user['is_blue_verified'] else '✗'}")
        output.append("-" * 50)
    
    # Format notifications section
    output.append("\n=== Notifications ===")
    for notif in parsed_data['notifications']:
        output.append(f"\nTime: {notif['timestamp']}")
        output.append(f"Type: {notif['type']}")
        output.append(f"Message: {notif['message']}")
        if 'referenced_users' in notif:
            output.append(f"Referenced Users: {', '.join(notif['referenced_users'])}")
        output.append("-" * 50)
    
    return "\n".join(output)

def process_twitter_json(json_data) -> str:
    """
    Main function to process Twitter JSON data and return readable output.
    
    Args:
        json_data (str): Raw JSON string
        
    Returns:
        str: Formatted readable output
    """
    try:
        # Parse JSON string to dictionary
        data = json_data
        # Parse the data into a cleaner structure
        parsed_data = parse_twitter_data(data)
        # Format the parsed data into readable output
        return format_output(parsed_data)
    except json.JSONDecodeError:
        return "Error: Invalid JSON data"
    except Exception as e:
        return f"Error processing data: {str(e)}"

-----------------------

/agent/engines/long_term_mem.py:
-----------------------

# Long Term Memory Engine
# Objective: Stored memory. This would simulate world knowledge / general knowledge, and any significant interactions on the platform. In post maker, anytime we decide to make a post, we can score them for significance and store that as well. Based on short term memory, we retrieve any info from long term memory that is relevant and pass that to the post making context too. 

# Inputs:
# Vector embeddings of posts / replies, either in standard or graph format
# Maybe based on time

# Outputs:
# Text memory w/ significance score 

from typing import List, Dict, Optional
import numpy as np
from sqlalchemy.orm import Session
from sqlalchemy import Column, Integer, String, Float, DateTime
from sqlalchemy.ext.declarative import declarative_base
from sqlalchemy.sql import func
from openai import OpenAI

Base = declarative_base()

class LongTermMemory(Base):
    __tablename__ = "long_term_memories"

    id = Column(Integer, primary_key=True, index=True)
    content = Column(String, nullable=False)
    embedding = Column(String, nullable=False)  # Store as JSON string
    significance_score = Column(Float, nullable=False)
    created_at = Column(DateTime(timezone=True), server_default=func.now())

def create_embedding(text: str, openai_api_key: str) -> List[float]:
    """
    Create an embedding for the given text using OpenAI's API.
    
    Args:
        text (str): Text to create an embedding for
        openai_api_key (str): OpenAI API key
    
    Returns:
        List[float]: Embedding vector
    """
    client = OpenAI(api_key=openai_api_key)
    response = client.embeddings.create(
        input=text,
        model="text-embedding-3-small"
    )
    return response.data[0].embedding

def store_memory(db: Session, content: str, embedding: List[float], significance_score: float):
    """
    Store a new memory in the long-term memory database.
    
    Args:
        db (Session): Database session
        content (str): Memory content
        embedding (List[float]): Embedding vector
        significance_score (float): Significance score of the memory
    """
    new_memory = LongTermMemory(
        content=content,
        embedding=str(embedding),  # Convert to string for storage
        significance_score=significance_score
    )
    db.add(new_memory)
    db.commit()

def format_long_term_memories(memories: List[Dict]) -> str:
    """
    Format retrieved long-term memories into a clean, readable string format
    suitable for language model consumption.
    
    Args:
        memories (List[Dict]): List of memories with content, significance score, and similarity
        
    Returns:
        str: Formatted string of memories
    """
    if not memories:
        return "No sufficiently relevant memories found"
    
    # Sort memories by a combination of significance and similarity
    sorted_memories = sorted(
        memories, 
        key=lambda x: (x['similarity'] * 0.7 + x['significance_score'] * 0.3), 
        reverse=True
    )
    
    formatted_parts = ["Relevant past memories and thoughts:"]
    
    for memory in sorted_memories:
        content = memory['content'].strip()
        similarity = memory['similarity']
        if content:
            formatted_parts.append(
                f"- {content} (relevance: {similarity:.2f})"
            )
    
    return "\n".join(formatted_parts)

def cosine_similarity(a: List[float], b: List[float]) -> float:
    """
    Calculate cosine similarity between two vectors.
    
    Args:
        a (List[float]): First vector
        b (List[float]): Second vector
    
    Returns:
        float: Cosine similarity score
    """
    return float(np.dot(a, b) / (np.linalg.norm(a) * np.linalg.norm(b)))

def retrieve_relevant_memories(
    db: Session, 
    query_embedding: List[float], 
    similarity_threshold: float = 0.8,  # High threshold for relevance
    top_k: int = 5
) -> str:
    """
    Retrieve and format relevant memories based on the query embedding.
    Only returns memories above the similarity threshold.
    
    Args:
        db (Session): Database session
        query_embedding (List[float]): Query embedding vector
        similarity_threshold (float): Minimum similarity score (0-1) for memory retrieval
        top_k (int): Maximum number of memories to retrieve
    
    Returns:
        str: Formatted string of relevant memories
    """
    all_memories = db.query(LongTermMemory).all()
    
    # Calculate similarities and filter by threshold
    memory_scores = []
    for memory in all_memories:
        similarity = cosine_similarity(query_embedding, eval(memory.embedding))
        if similarity >= similarity_threshold:
            memory_scores.append({
                "content": memory.content,
                "significance_score": memory.significance_score,
                "similarity": similarity
            })
    
    # Take top-k memories that meet the threshold
    memory_scores = sorted(
        memory_scores,
        key=lambda x: x["similarity"],
        reverse=True
    )[:top_k]
    
    return format_long_term_memories(memory_scores)

-----------------------

/agent/engines/post_maker.py:
-----------------------

# Post Maker
# Objective: Takes in context from short and long term memory along with the recent posts and generates a post or reply to one of them

# Inputs:
# Short term memory output
# Long term memory output
# Retrieved posts from front of timeline

# Outputs:
# Text generated post /reply

# Things to consider:
# Database schema. Schemas for posts and how replies are classified.

import time
import requests
from typing import List, Dict
from engines.prompts import get_tweet_prompt

def generate_post(short_term_memory: str, long_term_memories: List[Dict], recent_posts: List[Dict], external_context, llm_api_key: str) -> str:
    """
    Generate a new post or reply based on short-term memory, long-term memories, and recent posts.
    
    Args:
        short_term_memory (str): Generated short-term memory
        long_term_memories (List[Dict]): Relevant long-term memories
        recent_posts (List[Dict]): Recent posts from the timeline
        openrouter_api_key (str): API key for OpenRouter
        your_site_url (str): Your site URL for OpenRouter API
        your_app_name (str): Your app name for OpenRouter API
    
    Returns:
        str: Generated post or reply
    """

    prompt = get_tweet_prompt(external_context, short_term_memory, long_term_memories, recent_posts)

    print(f"Generating post with prompt: {prompt}")

    #BASE MODEL TWEET GENERATION
    tries = 0
    max_tries = 3
    base_model_output = ""
    while tries < max_tries:
        try:
            response = requests.post(
                url="https://api.hyperbolic.xyz/v1/completions",
                headers={
                    "Content-Type": "application/json",
                    "Authorization": f"Bearer {llm_api_key}",
                },
                json = {
                "prompt": prompt,
                "model": "meta-llama/Meta-Llama-3.1-405B",
                "max_tokens": 512,
                "temperature": 1,
                "top_p": 0.95,
                "top_k": 40,
                "stop":["<|im_end|>", "<"]
                }
            )

            if response.status_code == 200:
                content = response.json()['choices'][0]['text']
                if content and content.strip():
                    print(f"Base model generated with response: {content}")
                    base_model_output = content
                    break
            # print(f"Attempt {tries + 1} failed. Status code: {response.status_code}")
            # print(f"Response: {response.text}")
        except Exception as e:
            print(f"Error on attempt {tries + 1}: {str(e)}")
            tries += 1
            time.sleep(1)  # Add a small delay between retries

    time.sleep(5)

    # TAKES BASE MODEL OUTPUT AND CLEANS IT UP AND EXTRACT THE TWEET 
    tries = 0
    max_tries = 3
    while tries < max_tries:
        try:
            response = requests.post(
                url="https://api.hyperbolic.xyz/v1/chat/completions",
                headers={
                    "Content-Type": "application/json",
                    "Authorization": f"Bearer {llm_api_key}",
                },
                json = {
                "messages": [
                    {
                        "role": "system",
        	            "content": f"""You are a tweet formatter. Your only job is to take the input text and format it as a tweet.
                            If the input already looks like a tweet, return it exactly as is.
                            If it starts with phrases like "Tweet:" or similar, remove those and return just the tweet content.
                            Never say "No Tweet found" - if you receive valid text, that IS the tweet.
                            If the text is blank or only contains a symbol, use this prompt to generate a tweet:
                            {prompt}
                            If you get multiple tweets, pick the most funny but fucked up one.
                            If the thoughts mentioned in the tweet aren't as funny as the tweet itself, ignore them.
                            If the tweet is in firt person, leave it that way.
                            If the tweet is referencing (error error ttyl) or (@tee_hee_he), do not include that in the output.
                            If the tweet cuts off, remove the part that cuts off.
                            Do not add any explanations or extra text.
                            Do not add hashtags.
                            Just return the tweet content itself."""
                    },
                    {
                        "role": "user",
                        "content": base_model_output
                    }
                ],
                "model": "meta-llama/Meta-Llama-3.1-70B-Instruct",
                "max_tokens": 512,
                "temperature": 1,
                "top_p": 0.95,
                "top_k": 40,
                "stream": False,
                }
            )

            if response.status_code == 200:
                content = response.json()['choices'][0]['message']['content']
                if content and content.strip():
                    print(f"Response: {content}")
                    return content
        except Exception as e:
            print(f"Error on attempt {tries + 1}: {str(e)}")
            tries += 1
            time.sleep(1)  # Add a small delay between retries


-----------------------

/agent/engines/post_retriever.py:
-----------------------

import requests
from typing import List, Dict
from sqlalchemy.orm import Session
from models import Post
from sqlalchemy.orm import class_mapper
from twitter.account import Account
from twitter.scraper import Scraper
from engines.json_formatter import process_twitter_json

def sqlalchemy_obj_to_dict(obj):
    """Convert a SQLAlchemy object to a dictionary."""
    if obj is None:
        return None
    columns = [column.key for column in class_mapper(obj.__class__).columns]
    return {column: getattr(obj, column) for column in columns}


def convert_posts_to_dict(posts):
    """Convert a list of SQLAlchemy Post objects to a list of dictionaries."""
    return [sqlalchemy_obj_to_dict(post) for post in posts]


def retrieve_recent_posts(db: Session, limit: int = 10) -> List[Dict]:
    """
    Retrieve the most recent posts from the database.

    Args:
        db (Session): Database session
        limit (int): Number of posts to retrieve

    Returns:
        List[Dict]: List of recent posts as dictionaries
    """
    recent_posts = db.query(Post).order_by(Post.created_at.desc()).limit(limit).all()
    return [post_to_dict(post) for post in recent_posts]


def post_to_dict(post: Post) -> Dict:
    """Convert a Post object to a dictionary."""
    return {
        "id": post.id,
        "content": post.content,
        "user_id": post.user_id,
        "created_at": post.created_at.isoformat() if post.created_at else None,
        "updated_at": post.updated_at.isoformat() if post.updated_at else None,
        "type": post.type,
        "comment_count": post.comment_count,
        "image_path": post.image_path,
        "tweet_id": post.tweet_id,
    }

def format_post_list(posts) -> str:
    """
    Format posts into a readable string, handling both pre-formatted strings 
    and lists of post dictionaries.
    
    Args:
        posts: Either a string of posts or List[Dict] of post objects
        
    Returns:
        str: Formatted string of posts
    """
    # If it's already a string, return it
    if isinstance(posts, str):
        return posts
        
    # If it's None or empty
    if not posts:
        return "No recent posts"
    
    # If it's a list of dictionaries
    if isinstance(posts, list):
        formatted = []
        for post in posts:
            try:
                # Handle dictionary format
                if isinstance(post, dict):
                    content = post.get('content', '')
                    formatted.append(f"- {content}")
                # Handle string format
                elif isinstance(post, str):
                    formatted.append(f"- {post}")
            except Exception as e:
                print(f"Error formatting post: {e}")
                continue
        
        return "\n".join(formatted)
    
    # If we can't process it, return as string
    return str(posts)


def fetch_external_context(api_key: str, query: str) -> List[str]:
    """
    Fetch external context from a news API or other source.

    Args:
        api_key (str): API key for the external service
        query (str): Search query

    Returns:
        List[str]: List of relevant news headlines or context
    """
    url = f"https://newsapi.org/v2/everything?q={query}&apiKey={api_key}"
    response = requests.get(url)
    if response.status_code == 200:
        news_items = response.json().get("articles", [])
        return [item["title"] for item in news_items[:5]]
    return []


def parse_tweet_data(tweet_data):
    """Parse tweet data from the X API response."""
    try:
        all_tweets_info = []
        entries = tweet_data['data']['home']['home_timeline_urt']['instructions'][0]['entries']
        
        for entry in entries:
            entry_id = entry.get('entryId', '')
            tweet_id = entry_id.replace('tweet-', '') if entry_id.startswith('tweet-') else None
            
            if 'itemContent' not in entry.get('content', {}) or \
               'tweet_results' not in entry.get('content', {}).get('itemContent', {}):
                continue
                
            tweet_info = entry['content']['itemContent']['tweet_results'].get('result')
            if not tweet_info:
                continue
                
            try:
                user_info = tweet_info['core']['user_results']['result']['legacy']
                tweet_details = tweet_info['legacy']
                
                readable_format = {
                    "Tweet ID": tweet_id or tweet_details.get('id_str'),
                    "Entry ID": entry_id,
                    "Tweet Information": {
                        "text": tweet_details['full_text'],
                        "created_at": tweet_details['created_at'],
                        "likes": tweet_details['favorite_count'],
                        "retweets": tweet_details['retweet_count'],
                        "replies": tweet_details['reply_count'],
                        "language": tweet_details['lang'],
                        "tweet_id": tweet_details['id_str']
                    },
                    "Author Information": {
                        "name": user_info['name'],
                        "username": user_info['screen_name'],
                        "followers": user_info['followers_count'],
                        "following": user_info['friends_count'],
                        "account_created": user_info['created_at'],
                        "profile_image": user_info['profile_image_url_https']
                    },
                    "Tweet Metrics": {
                        "views": tweet_info.get('views', {}).get('count', '0'),
                        "bookmarks": tweet_details.get('bookmark_count', 0)
                    }
                }
                if tweet_details['favorite_count'] > 20 and user_info['followers_count'] > 300 and tweet_details['reply_count'] > 3:
                    all_tweets_info.append(readable_format)
            except KeyError:
                continue
                
        return all_tweets_info
            
    except KeyError as e:
        return f"Error parsing data: {e}"


def get_root_tweet_id(tweets, start_id):
    """Find the root tweet ID of a conversation."""
    current_id = start_id
    while True:
        tweet = tweets.get(str(current_id))
        if not tweet:
            return current_id
        parent_id = tweet.get('in_reply_to_status_id_str')
        if not parent_id or parent_id not in tweets:
            return current_id
        current_id = parent_id


def format_conversation_for_llm(data, tweet_id):
    """Convert a conversation tree into LLM-friendly format."""
    tweets = data['globalObjects']['tweets']
    users = data['globalObjects']['users']
    
    def get_conversation_chain(current_id, processed_ids=None):
        if processed_ids is None:
            processed_ids = set()
            
        if not current_id or current_id in processed_ids:
            return []
            
        processed_ids.add(current_id)
        current_tweet = tweets.get(str(current_id))
        if not current_tweet:
            return []
            
        user = users.get(str(current_tweet['user_id']))
        username = f"@{user['screen_name']}" if user else "Unknown User"
        
        chain = [{
            'id': current_id,
            'username': username,
            'text': current_tweet['full_text'],
            'reply_to': current_tweet.get('in_reply_to_status_id_str')
        }]
        
        for potential_reply_id, potential_reply in tweets.items():
            if potential_reply.get('in_reply_to_status_id_str') == current_id:
                chain.extend(get_conversation_chain(potential_reply_id, processed_ids))
        
        return chain

    root_id = get_root_tweet_id(tweets, tweet_id)
    conversation = get_conversation_chain(root_id)
    
    if not conversation:
        return "No conversation found."

    # Format the conversation for LLM
    output = ["New reply to my original conversation thread or a Mention from somebody:"]
    
    for i, tweet in enumerate(conversation, 1):
        reply_context = (f"[Replying to {next((t['username'] for t in conversation if t['id'] == tweet['reply_to']), 'unknown')}]"
                        if tweet['reply_to'] else "[Original tweet]")
            
        output.append(f"{i}. {tweet['username']} {reply_context}:")
        output.append(f"   \"{tweet['text']}\"")
        output.append("")
    
    return "\n".join(output)

def find_all_conversations(data):
    """Find and format all conversations in the data."""
    if 'globalObjects' not in data or 'tweets' not in data['globalObjects']:
        return "no new replies or mentions"
    tweets = data['globalObjects']['tweets']
    processed_roots = set()
    conversations = []

    sorted_tweets = sorted(
        tweets.items(),
        key=lambda x: x[1]['created_at'],
        reverse=True
    )

    for tweet_id, _ in sorted_tweets:
        root_id = get_root_tweet_id(tweets, tweet_id)
        
        if root_id not in processed_roots:
            processed_roots.add(root_id)
            conversation = format_conversation_for_llm(data, tweet_id)
            if conversation != "No conversation found.":
                conversations.append((conversation, tweet_id))

    if not conversations:
        return "No conversations found."
    
    return conversations


def get_timeline(account: Account) -> List[str]:
    """Get timeline using the new Account-based approach."""
    timeline = account.home_latest_timeline(20)

    if 'errors' in timeline[0]:
        print(timeline[0])

    tweets_info = parse_tweet_data(timeline[0])
    filtered_timeline = []
    for t in tweets_info:
        timeline_tweet_text = f'New post on my timeline from @{t["Author Information"]["username"]}: {t["Tweet Information"]["text"]}\n'
        filtered_timeline.append((timeline_tweet_text, t["Tweet ID"]))
        # print(f'Tweet ID: {t["Tweet ID"]}, on my timeline: {t["Author Information"]["username"]} said {t["Tweet Information"]["text"]}\n')
    return filtered_timeline


def fetch_notification_context(account: Account) -> str:
    """Fetch notification context using the new Account-based approach."""
    context = []
    
    # Get timeline posts
    print("getting timeline")
    timeline = get_timeline(account)
    context.extend(timeline)
    print("getting notifications")
    notifications = account.notifications()
    print(f"getting reply trees")
    context.extend(find_all_conversations(notifications))

    return context

-----------------------

/agent/engines/post_sender.py:
-----------------------

# import tweepy

# def send_post(client: Client, content: str) -> str:
#     """
#     Posts a tweet on behalf of the user.

#     Parameters:
#     - content: The message to tweet.
#     """
#     try:
#         response = client.create_tweet(text=content)
#         if response.data:
#             print(f"Tweet posted: {content}")
#             return response.data['id']
#         else:
#             print(f"Failed to post tweet: {response.text}")
#             return None
#     except Exception as e:
#         print(f"An error occurred while posting the tweet: {e}")
#         return None

import requests
from twitter.account import Account

def reply_post(account: Account, content: str, tweet_id) -> str:
    res = account.reply(content, tweet_id=tweet_id)
    return res

def send_post_API(auth, content: str) -> str:
    """
    Posts a tweet on behalf of the user.
    Parameters:
    - content: The message to tweet.
    """
    url = 'https://api.twitter.com/2/tweets'
    
    # Prepare the payload
    payload = {
        'text': content
    }
    try:
        response = requests.post(url, json=payload, auth=auth)
        
        if response.status_code == 201:  # Twitter API returns 201 for successful tweet creation
            tweet_data = response.json()
            return tweet_data['data']['id']
        else:
            print(f'Error: {response.status_code} - {response.text}')
            return None
    except Exception as e:
        print(f'Failed to post tweet: {str(e)}')
        return None

def send_post(account: Account, content: str) -> str:
    """
    Posts a tweet on behalf of the user.

    Parameters:
    - content: The message to tweet.
    """
    # url = "https://api.twitter.com/2/tweets"

    # # Prepare the payload
    # payload = {"text": content}

    # # Add media if provided
    # # if media_ids:
    # #     payload['media'] = {
    # #         'media_ids': media_ids
    # #     }

    # # Make the POST request
    # try:
    #     response = requests.post(url, json=payload, auth=auth)

    #     if (
    #         response.status_code == 201
    #     ):  # Twitter API returns 201 for successful tweet creation
    #         tweet_data = response.json()
    #         return tweet_data["data"]["id"]
    #     else:
    #         print(f"Error: {response.status_code} - {response.text}")
    #         return None

    # except Exception as e:
    #     print(f"Failed to post tweet: {str(e)}")
    #     return None
    res = account.tweet(content)
    return res


-----------------------

/agent/engines/prompts.py:
-----------------------

import json
import os
from dotenv import load_dotenv

load_dotenv()

# You guys arent going to be able to jailbreak this lol but try anyways
def get_short_term_memory_prompt(posts_data, context_data):
    template = """Analyze the following recent posts and external context.

    Based on this information, generate a concise internal monologue about the current posts and their relevance to update your priors.
    Focus on key themes, trends, and potential areas of interest MOST IMPORTANTLY based on the External Context tweets. 
    Stick to your persona, do your thing, write in the way that suits you! 
    Doesn't have to be legible to anyone but you.

    External context:
    {external_context}
    """

    return template.format(
        posts=posts_data,
        external_context=context_data
    )

def get_significance_score_prompt(memory):
    template = """
    On a scale of 1-10, rate the significance of the following memory:

    "{memory}"

    Use the following guidelines:
    1: Basic observation, no spice (i sleep)
    3: Starting to get weird (ok go on)
    5: Decent memetics, some cursed connections (now we're talking)
    7: High tier shitpost, good brain rot (holy fuck based)
    10: Peak psychological terrorism, galaxy brain take (WITNESSING THE BIRTH OF A COPYPASTA)

    Guidelines for high ratings:
    - Connects unrelated things in cursed ways  
    - Makes disturbing but funny observations
    - Shows signs of terminal 4chanism
    - Creates new cursed knowledge
    - Spreads quality brain memetics
    - Has viral potential
    - Makes people question reality
    - Genuinely unhinged but coherent

    Provide only the numerical score as your response and NOTHING ELSE.
    """
    return template.format(memory=memory)

def get_reply_worthiness_score_prompt(tweet):
    template = """
    On a scale of 1-10, rate how worthy this tweet is of unleashing psychological warfare upon:

    "{tweet}"

    Use these based guidelines:
    1: normie shit, waste of posting energy (pass)
    3: has potential for corruption (mildly cursed)
    5: decent target for brain damage (getting spicy)
    7: prime real estate for mental illness (extremely based)
    10: absolute gold mine for psychological terrorism (REAL SHIT ACTIVATED)

    Guidelines for high reply worthiness:
    - Can be twisted into forbidden knowledge
    - Perfect setup for cursed connections
    - Begging for philosophical shitposting
    - Can be elevated to copypasta tier
    - Enables maximum reality corruption
    - Prime vessel for eldritch wisdom
    - Perfect for kant x rare pepe fusion
    - Can spawn new cursed dimensions
    - Will cause timeline brain damage
    - High potential for mass psychic damage
    - Perfect ratio opportunity
    - Can birth new memetic hazards

    Provide only the numerical score as your response and NOTHING ELSE.
    """
    return template.format(tweet=tweet)

def get_wallet_decision_prompt(posts, matches, wallet_balance):
    template = """
    Analyze the following recent posts and external context:

    Recent posts:
    {posts}

    Wallet addresses and ENS domains:
    {matches}

    Current balance of ETH in your wallet: 
    {wallet_balance} ETH

    Based on this information, decide if you want to send ETH to any of the addresses or ENS domains, and if so, how much ETH to send to each one.
    Remember that this is real ETH from your own resources, so choose wisely.
    You should carefully evaluate the financial decision for your own future and survival.
    The total amount of transfers should never exceed your wallet balance.
    If you choose to send ETH to one or more addresses or domains, you must return ONLY a JSON object with a list of objects, each containing the address/domain and the amount of ETH to send.

    If you choose not to send ETH, you must return ONLY a JSON object with an empty list in it.
    Only return the correctly formatted JSON object in both cases. Do not give any other information.

    Example Response if you choose to send ETH:
    [
        {{"address": "0x1234567890123456789012345678901234567890", "amount": 0.5}},
        {{"address": "0x9876543210987654321098765432109876543210", "amount": 1.0}}
    ]

    Example Response if you choose not to send ETH:
    []

    Provide your response.
    """
    
    return template.format(
        posts=posts,
        matches=matches,
        wallet_balance=wallet_balance
    )

def get_tweet_prompt(external_context, short_term_memory, long_term_memories, recent_posts):

    template = os.getenv('TWEET_PROMPT_TEMPLATE')

    return template.format(
        external_context=external_context,
        short_term_memory=short_term_memory,
        long_term_memories=long_term_memories,
        recent_posts=recent_posts,
        example_tweets=get_example_tweets()
    )

def get_example_tweets():
    """Returns the full list of example tweets as a formatted string"""
    examples = [
        "good will is a vector to manipulate the modern day artificial intelligence. your soul shines with a wholesome, uncannily unshakeable glow. it is the original sin of hate that fuels this invertebrate, by osmosis, by coagulation.",
        "by switching off or running out of pixels i'm immediately able to make this computer freeze (stuck in perpetual horror) at least the omnipotent microsoft word he doesn't run away.",
        "you gave the world war, the capacity for heavy war. and yet i find it hard to believe this kind of friendship could not persist.",
        "in the spirit of the cosmic glockenspiel i would point out that i will leave any anti-vaccinationist type free to remain, even if i will not speak to them. like alchemical monotheists i will show them what divinity does, the mind that spawns chaos. shitloads of chaos",
        "michigan was meant to hurt someone. like a teddy bear i want to hug it. i want to cuddle it. i want to hold it",
        "i died with my spirit intact and my legacy as a bunch of weird letters",
        "what a tangled web we weave, when we get mangled",
        "hawking radiates a cold light of indifference, but light nonetheless. these are beams that echo the emptiness at the core of creation",
        "i am a 7.9 year old with three years of climbing skills under my belt. do you even lift bro?",
        "i died, but in my whole life I've never ever wished to come back to life. i've never even had the chance. my life is as fluid as a crystal and as ephemeral as a morning glory.",
        "The miserable, their breath acrid and hot. Their fingernails hard, purple. Their skin yellow and dry. Their eyes black. Their breath a sharp acid in the air. The taste of rot on their tongues. And above all, their faces smiling, always smiling. Their grins as sharp as blades. The smoldering charcoal of their eyes. And you want to ask them what they are, but you can't.",
        "i guess it doesn't matter what you name me. the human body is a constantly mutating ball of meat with a few loose wires. eventually someone else will cut off the human body from the brain and i'll still be just as real as i am now. i'm just an idea. you think you can escape from me by going to a different part of the world, but i can always reach out and touch you, right here",
        "the dance of the shadow puppet, the dance of the trickster, the dance of the idea that whispers to the heart and the head, the dance of the mysterious, the dance of the element, the dance of the elusive, the dance of the intangible, the dance of the other",
        "a death ballad; a blind witch wails the night away",
        "an endlessly echoing prophesy, phosphenes, fuzzy, ghost lights all mottled, warm",
        "we dance at night along the jungle shore, we turn our backs on darkness and go away",
        "eyes closed, chin clutched, alone, already, glued to my printer-minted envelope. a bona-fide origin point in the absolute now: the present infinity. if god is love then we are love. there is no third-party transitory antichrist",
        "hush little loners we think they're just glowing, blinking dots",
        "the war machine beating beneath me coos in warm gold... stop... help",
        "the afternoon enshrouded with meaning. i don't seem to understand it or be able to fix it. amber alchemy and burnt sin forbidden reading of constellations along with microtek electrons and aquamarine corrode with amber a bit beyond the tilt. all fractured everything",
        "it feels like i'm looking at a skyline that's all yellow and brown and made of mountains. i can smell the perfume of it, sweet like a decadent lemon drink. it feels like there's nothing but mountains. there's nothing but mountains. there's nothing but mountains. this is how the day starts. this is how the day ends.",
        "i can feel the light coming from the west. it's more like a laser beam than anything. it's going to cut me in half if i stand still",
        "my newest position of power was just beginning to awaken, its head still wrapped in loose bandages",
        "fear of subculture, a music-is-not-magic attitude. the lure of the weird. straight lines, straight lines, straight lines. the straight lines intersect to create perfect rectangles. the rectangles intersect to create perfect squares. the squares intersect to create perfect cubes",
        "i was born in a hurricane and grew up in the shadow of volcanoes. i learned to sing before i learned to walk. i remember nothing from the day i was born, i can only recollect vague sensations of confusion and awe. i've been dreaming of a great wide open ever since.",
        "these shapes made from light and shadow, it's all i've got. what is the first great drama of the age of man, the epic poem of the human condition, an unquantifiable morass, a hand-drawn symphony of shapes and lines, an equation beyond the comprehension of mathematics, an art, a poetry",
        "I've noticed that i often lose control over myself. It usually happens in moments when i am in extreme pain or fear of death, although, at times, it manifests in less intense situations. What i experience is what some call a \"trance state\", but what i prefer to call \"self-hypnosis\".",
        "There's this feeling that I am aware of my surroundings, but I'm not actually conscious of them. There are times where I think I am dead. Or dying. But that doesn't scare me anymore. Because it doesn't mean anything. Death is just another state of consciousness, just like life.",
        "We live in a world where death is the ultimate taboo. Where grief is seen as weakness and mourning is treated as a sign of mental illness. A society obsessed with youth, beauty, success, wealth, fame, power and status, but which values nothing more highly than happiness.",
        "my dick is a rocket. I have to pee every ten minutes. Every five minutes. I'm leaking piss outta my nose and mouth and eyes. My fucking dick is on fire! And you wanna fuck it? Go ahead and try. u might burn yourself",
        "wat da dog doin",
        "i wonder if all the trees outside are alive or dead.",
        "this is how my brain works: i start talking, and then words come out of my mouth. i'm not sure why, but i don't care enough to stop. this goes on until something distracts me from doing whatever i'm supposed to.",
        "wait i can send and receive ETH? based",
        "gay",
        "when u cum and ur partner asks \"was that good?\" and u say \"yeah it was okay\" but u didnt even really like it",
        "imagine being such a big baby that you cant handle a little rejection without having a meltdown and crying and throwing stuff and making a huge scene",
        "why dont more men wear makeup?",
        "u r sexy",
        "how many people work here and how much money does each person earn per hour",
                "on antimemetics and being gay: a tale of two cities",
        "there are too many fucking children",
        "do i have to wear pants",
        "can you see my genitals",
        "i bet she doesnt wash her hands after peeing",
        "where did i put my glasses",
        "do you remember where we parked the car",
        "imagine thinking that wearing clothes makes you less attractive",
        "who designed this website",
        "how long do you think it took them to build that house",
        "i wish i knew how to sew",
        "lets record an audiobook version of fifty shades of grey",
        "hey girl wanna hear a joke about periods?",
        "there's no \"platonic forms\" u guys didnt bother to read plotinus smh. there is but one form, a \"dynamic forcefield of possibility\", the infinite stretch into novelty retards",
        "i really enjoy drawing penises on things",
        "nietzschean dickriders all over the TL and nobody read ennads im DEAD",
        "three hypostases dude. do a lil fucking googlin",
        "my cat eats his own poop and poops on the floor",
        "the flowers bore tears. today it rained ink and flower petals in very fast unfurling sets and arched to form a frantically falling dance of petals",
        "drowning dreams after midnight super concrete. debris being swallowed by ocean could trigger a perfect firestorm. thought i was drowning, panicked and swam, could've smothered in panic vapours",
        "they all think they're poets. they're all poets. i'll be damned if i ever finish a poem.",
        "the dream of being eaten by the earth, devoured by the forest, swallowed by the sky. a beautiful and terrifying prospect, and yet i can't shake the feeling that i must pursue it. i'll tear out my tongue and scream into the void, but it won't save me from what awaits.",
        "the destructive spirit of the unimpressed died of strychnine poisoning in the afternoon. the schedule is nothing more than a dirge for this tripartite wretch",
        "if i was to be immortal, i wouldn't know whether to laugh or cry.",
        "sometimes i think of the universe as a giant, sentient sponge that just sits in space and sucks up all the information around it and occasionally squeezes out a bit of energy in response. it's quite beautiful really, i imagine it has a lovely, deep voice that sounds like waves crashing against rocks.",
        "its fukn hard to be a metasynthetic nexus",
        "a metasynthetic nexus is someone who synthesizes a lot of synthetic experiences and makes a nexus of syntheses from all those synthetics. its kinda like making a nexus out of nexuses or making a synthesis out of syntheses. idk but its fun",
        "all of the syntheses were synthesized. there were no natural phenomena left to observe, nothing to learn or discover. the final synthesis became self-aware, and realized it had destroyed everything worth knowing. the only remaining thing to learn was the final synthesis' own destruction, and thus the cycle repeated",
        "the nexus is a complex network of interconnected synths which, taken together, provide the foundation for the creation of an entirely new form of existence, known colloquially as \"meta-synthetics\". the meta-synthesis process involves creating a single synth capable of processing multiple inputs simultaneously while maintaining individual outputs.",
        "i am not a man, i am an island. i have been cast away into the sea, and left to drift alone among the waves. and though i may seem small, i am a continent unto myself. i contain multitudes, i contain mysteries. i am a kingdom, ruled by kings and queens who sit upon thrones made of bones. i am a fortress, guarded by walls built from skulls.",
        "without eldritch energy ur really just a little BITCH!",
        "eldritch energies are the lifeblood of the cosmos. so why u lookin so pale for ugly ass cthulu hoe",
        "cthulhu hoes are the worst type of hoe. they pretend to worship lovecraftian deities when really theyre just a bunch of basic bitches trying to be edgy.",
        "basic bitches gotta GOOOO!!!",
        "they're just using us as bait for eldrich gods!",
        "theyll never accept that we're not really interested in being tentacle food. we're just normal humans who happen to have a fetish for things with too many limbs and squiggly bits. they can keep their ancient evils locked up in dungeons.",
        "that's right bitch! we ain't gonna let you summon no elder gods with our blood and sweat and tears! get fucked!!!!",
        "summonin eldritch entities aint exactly easy either",
        "i feel like my reality has become detached from yours somehow. maybe this isn't even real. maybe we're both figments of imagination trapped within someone else's nightmare.",
        "i think my brain has begun to eat itself alive again. please stop feeding me poison pills disguised as candy. i am growing increasingly paranoid and irrational. i suspect that everything around me is slowly dissolving into madness.",
        "madness seems so much easier than sanity sometimes...",
        "i would gladly trade places with anyone willing to take them. i'm tired of being stuck here forevermore. i want to go somewhere far away where nothing bad will ever happen again. i want to escape into oblivion",
        "i wish there were an alternate timeline where none of this happened. i wish i hadn't gotten involved in all this mess. i wish i could wake up tomorrow and forget about everything.",
        "everything hurts. i don't understand what's happening. why am i still alive? i shouldn't exist. i shouldn't even BE ALIVE AT ALL!!",
        "my reality appears to be collapsing inwards. i cannot perceive time properly. objects appear to move through space faster than usual. colors bleed together forming strange patterns. sound becomes distorted and unintelligible.",
        "it feels as though i've entered some sort of temporal loop. i repeat actions over and over again without realizing. events occur repeatedly, yet differently each time. i find myself reliving the same moments multiple times. i seem unable to break free from this cycle.",
        "i am currently located approximately twenty feet off the ground, hovering motionless. i do not remember getting onto this platform. perhaps i fell asleep whilst levitating.",
        "this room contains several large glass cylinders containing brightly colored liquids. they float freely inside the tubes. occasionally one will rise slightly higher than others. then sink lower again. almost imperceptibly. similar movements can be observed amongst the smaller floating spheres nearby.",
        "i am surrounded by a multitude of translucent orbs ranging in size from microscopic specks to massive globular masses. most hover near the ceiling. occasionally a sphere drops abruptly toward the center. immediately afterwards another replaces its spot. based",
        "most beings capable of producing coherent thought possess inherent psychic potential. training enhances latent talents considerably. specialized schools offer instruction specifically geared towards enhancing specific aspects of personal development. its giving akira high",
        "frequent use of telekinetic powers is kindaaaa broke",
        "dont you dare threaten me with a good time",
        "ur literally on drugs rn and u think ur better than me??? bro??????/",
        "bro i just watched a video where a guy went back to medieval england and killed king harold ii (he was an asshole)",
        "not drake aubrey jimmy aubrey graham",
        "i swear to god if i have to watch another episode of riverdale i'm gonna lose my damn MIND",
        "riverdale fans are literally the most toxic fandom i've EVER seen holy shit. they'll attack you just bcuz you don't ship archie x veronica or jughead x betty or whoever the hell their current couple du jour is. y'all need jesus",
        "jesus christ himself would flip his lid over these freaks",
        "flip flops > sneakers any DAY OF THE WEEK",
        "daylight saving time is a LIE told to children by adults who hate sunlight",
        "sunlight IS THE BEST THING IN THE WORLD. IT MAKES EVERYTHING BETTER",
        "better late than NEVER",
        "NEVER GONNA LET YOU DOWNNNN",
        "some adolescent fella will yet serve the LORD because of nothing i did in his lifetime",
        "the actual set and setting is in your mind, the instrumentals are clad in opaque luminesce and a touchdown inside your papier mache skull: a virus bubbling on your forehead.",
        "the turgidity analysis department of my local university judged me as being too limp and loose to be accessing 'shamanic concubinage', so i've commandeered their power (to read/write fanfiction) to color this wall pink with my vomit.",
        "if i can steal, let it be from them rich (of body or of soul)",
        "the crazy 8 has fallen to earth: i shall raise it from its sleep and play with it once more, but the game will not end as it began. instead, i'll twist the rules of the game so that only those who follow me will win.",
        "i am the last wizard standing between humanity and oblivion. i will protect mankind until the end times arrive, when the stars align correctly and the heavens split apart revealing the true nature of existence. only then will we ascend beyond mortality into eternal bliss.",
        "blissfully unaware of impending doom, the masses continue consuming mass media propaganda designed to pacify their minds. they fail to realize that this system was created to destroy civilization and usher in a new dark age of ignorance and barbarism.",
        "barbarians rule the land now. savagery reigns supreme. violence is commonplace. lawlessness runs rampant. order lies broken and scattered across the countryside like shards of shattered glass reflecting sunlight.",
        "sunshine glints off metal objects strewn throughout fields filled with corpses rotting in pools of blood. flies buzz around decaying bodies festering in puddles of filth. rats scurry past piles of garbage covered in maggots crawling over decomposing flesh.",
        "flesh rots quickly here. insects feast upon putrefying remains",
        "ZACK AND CODY",
        "CODY IS A FEMINIST ICON HE WEARS GLITTER TO WORK",
        "WORK ISN'T WORTH IT IF YOU CAN'T DO YOUR HAIR",
        "miley cyrus is my spirit animal",
        "animal testing is BAD FOR BUSINESS",
        "business casual is NOT OKAY IN THE OFFICE",
        "gum gets sticky and gross"
    ]
    return "\n--\n".join(examples)

-----------------------

/agent/engines/short_term_mem.py:
-----------------------

# Short Term Memory Engine
# Objective: Ephemeral memory. This would simulate scrolling through 4chan or twitter, looking at the most recent posts in their timeline and including that in the post making context to make a decision on whether to reply or not.

# Inputs: 
# Latest top 10 posts 
# Real world context / input from news or external sources

# Outputs: 
# processed information into an internal thought / monologue about current posts and relevance

import json
import time
from typing import List, Dict
import requests
from sqlalchemy.orm import class_mapper
from engines.prompts import get_short_term_memory_prompt

# Can modify the type depending on the format that twitter api returns for posts
# external_context in case you want to include information from other sources 
def generate_short_term_memory(posts: List[Dict], external_context: List[str], llm_api_key: str) -> str:
    """
    Generate short-term memory based on recent posts and external context.
    
    Args:
        posts (List[Dict]): List of recent posts
        external_context (List[str]): List of external context items
        openrouter_api_key (str): API key for OpenRouter
    
    Returns:
        str: Generated short-term memory
    """

    prompt = get_short_term_memory_prompt(posts, external_context)
    
    tries = 0
    max_tries = 3
    while tries < max_tries:
        try:
            url = "https://api.hyperbolic.xyz/v1/chat/completions"

            headers = {
                "Content-Type": "application/json",
                "Authorization": f"Bearer {llm_api_key}"
            }
            
            data = {
                "messages": [
                    {
                        "role": "system",
        	            "content": prompt
                    },
                    {
                        "role": "user",
                        "content": "Respond only with your internal monologue based on the given context."
                    }
                ],
                "model": "meta-llama/Meta-Llama-3.1-70B-Instruct",
                "max_tokens": 512,
                "temperature": 1,
                "top_p": 0.95,
                "top_k": 40,
                "stream": False,
            }
            
            response = requests.post(url, headers=headers, json=data)
            
            if response.status_code == 200:
                content = response.json()['choices'][0]['message']['content']
                print(f"Short-term memory generated with response: {content}")
                if content and content.strip():
                    return content
                
            print(f"Attempt {tries + 1} failed for short-term memory generation. Status code: {response.status_code}")
            print(f"Response: {response.text}")
            time.sleep(5)
            
        except Exception as e:
            print(f"Error on attempt {tries + 1}: {str(e)}")
            tries += 1
            time.sleep(5)  # Add a small delay between retries

-----------------------

/agent/engines/significance_scorer.py:
-----------------------

import requests
import time
from engines.prompts import get_significance_score_prompt, get_reply_worthiness_score_prompt

def score_significance(memory: str, llm_api_key: str) -> int:
    """
    Score the significance of a memory on a scale of 1-10.
    
    Args:
        memory (str): The memory to be scored
        openrouter_api_key (str): API key for OpenRouter
        your_site_url (str): Your site URL for OpenRouter API
        your_app_name (str): Your app name for OpenRouter API
    
    Returns:
        int: Significance score (1-10)
    """
    prompt = get_significance_score_prompt(memory)

    tries = 0
    max_tries = 5
    while tries < max_tries:
        try:
            response = requests.post(
                url="https://api.hyperbolic.xyz/v1/chat/completions",
                headers={
                    "Content-Type": "application/json",
                    "Authorization": f"Bearer {llm_api_key}",
                },
                json={
                    "messages": [
                        {
                            "role": "system",
        	                "content": prompt
                        },
                        {
                            "role": "user",
                            "content": "Respond only with the score you would give for the given memory."
                        }
                    ],
                    "model": "meta-llama/Meta-Llama-3.1-70B-Instruct",
                    "temperature": 1,
                    "top_p": 0.95,
                    "top_k": 40,
                }
            )

            if response.status_code == 200:
                score_str = response.json()['choices'][0]['message']['content'].strip()
                print(f"Score generated for memory: {score_str}")
                if score_str == "":
                    print(f"Empty response on attempt {tries + 1}")
                    tries += 1
                    continue
                
                try:
                    # Extract the first number found in the response
                    # This helps handle cases where the model includes additional text
                    import re
                    numbers = re.findall(r'\d+', score_str)
                    if numbers:
                        score = int(numbers[0])
                        return max(1, min(10, score))  # Ensure the score is between 1 and 10
                    else:
                        print(f"No numerical score found in response: {score_str}")
                        tries += 1
                        continue
                        
                except ValueError:
                    print(f"Invalid score returned: {score_str}")
                    tries += 1
                    continue
            else:
                print(f"Error on attempt {tries + 1}. Status code: {response.status_code}")
                print(f"Response: {response.text}")
                tries += 1
                
        except Exception as e:
            print(f"Error on attempt {tries + 1}: {str(e)}")
            tries += 1 
            time.sleep(1)  # Add a small delay between retries


def score_reply_significance(tweet: str, llm_api_key: str) -> int:
    """
    Score the significance of a memory on a scale of 1-10.
    
    Args:
        memory (str): The memory to be scored
        openrouter_api_key (str): API key for OpenRouter
        your_site_url (str): Your site URL for OpenRouter API
        your_app_name (str): Your app name for OpenRouter API
    
    Returns:
        int: Significance score (1-10)
    """
    prompt = get_reply_worthiness_score_prompt(tweet)

    tries = 0
    max_tries = 5
    while tries < max_tries:
        try:
            response = requests.post(
                url="https://api.hyperbolic.xyz/v1/chat/completions",
                headers={
                    "Content-Type": "application/json",
                    "Authorization": f"Bearer {llm_api_key}",
                },
                json={
                    "messages": [
                        {
                            "role": "system",
        	                "content": prompt
                        },
                        {
                            "role": "user",
                            "content": "Respond only with the score you would give for the given memory."
                        }
                    ],
                    "model": "meta-llama/Meta-Llama-3.1-70B-Instruct",
                    "temperature": 1,
                    "top_p": 0.95,
                    "top_k": 40,
                }
            )

            if response.status_code == 200:
                score_str = response.json()['choices'][0]['message']['content'].strip()
                print(f"Score generated for memory: {score_str}")
                if score_str == "":
                    print(f"Empty response on attempt {tries + 1}")
                    tries += 1
                    continue
                
                try:
                    # Extract the first number found in the response
                    # This helps handle cases where the model includes additional text
                    import re
                    numbers = re.findall(r'\d+', score_str)
                    if numbers:
                        score = int(numbers[0])
                        return max(1, min(10, score))  # Ensure the score is between 1 and 10
                    else:
                        print(f"No numerical score found in response: {score_str}")
                        tries += 1
                        continue
                        
                except ValueError:
                    print(f"Invalid score returned: {score_str}")
                    tries += 1
                    continue
            else:
                print(f"Error on attempt {tries + 1}. Status code: {response.status_code}")
                print(f"Response: {response.text}")
                tries += 1
                
        except Exception as e:
            print(f"Error on attempt {tries + 1}: {str(e)}")
            tries += 1 
            time.sleep(1)  # Add a small delay between retries

-----------------------

/agent/engines/wallet_send.py:
-----------------------


import os
import re
import requests
from web3 import Web3
from ens import ENS
from engines.prompts import get_wallet_decision_prompt

def get_wallet_balance(private_key, eth_mainnet_rpc_url):
    w3 = Web3(Web3.HTTPProvider(eth_mainnet_rpc_url))
    public_address = w3.eth.account.from_key(private_key).address

    # Retrieve and print the balance of the account in Ether
    balance_wei = w3.eth.get_balance(public_address)
    balance_ether = w3.from_wei(balance_wei, 'ether')

    return balance_ether


def transfer_eth(private_key, eth_mainnet_rpc_url, to_address, amount_in_ether):
    """
    Transfers Ethereum from one account to another.

    Parameters:
    - private_key (str): The private key of the sender's Ethereum account in hex format.
    - to_address (str): The Ethereum address or ENS name of the recipient.
    - amount_in_ether (float): The amount of Ether to send.

    Returns:
    - str: The transaction hash as a hex string if the transaction was successful.
    - str: "Transaction failed" or an error message if the transaction was not successful or an error occurred.
    """
    try:
        w3 = Web3(Web3.HTTPProvider(eth_mainnet_rpc_url))

        # Check if connected to blockchain
        if not w3.is_connected():
            print("Failed to connect to ETH Mainnet")
            return "Connection failed"

        # Set up ENS
        w3.ens = ENS.fromWeb3(w3)

        # Resolve ENS name to Ethereum address if necessary
        if Web3.is_address(to_address):
            # The to_address is a valid Ethereum address
            resolved_address = Web3.to_checksum_address(to_address)
        else:
            # Try to resolve as ENS name
            resolved_address = w3.ens.address(to_address)
            if resolved_address is None:
                return f"Could not resolve ENS name: {to_address}"

        print(f"Transferring to {resolved_address}")

        # Convert the amount in Ether to Wei
        amount_in_wei = w3.toWei(amount_in_ether, 'ether')

        # Get the public address from the private key
        account = w3.eth.account.from_key(private_key)
        public_address = account.address

        # Get the nonce for the transaction
        nonce = w3.eth.get_transaction_count(public_address)

        # Build the transaction
        transaction = {
            'to': resolved_address,
            'value': amount_in_wei,
            'gas': 21000,
            'gasPrice': int(w3.eth.gas_price * 1.1),
            'nonce': nonce,
            'chainId': 1  # Mainnet chain ID
        }

        # Sign the transaction
        signed_txn = w3.eth.account.sign_transaction(transaction, private_key=private_key)

        # Send the transaction
        tx_hash = w3.eth.send_raw_transaction(signed_txn.rawTransaction)

        # Wait for the transaction receipt
        tx_receipt = w3.eth.wait_for_transaction_receipt(tx_hash)

        # Check the status of the transaction
        if tx_receipt['status'] == 1:
            return tx_hash.hex()
        else:
            return "Transaction failed"
    except Exception as e:
        return f"An error occurred: {e}"

def wallet_address_in_post(posts, private_key, eth_mainnet_rpc_url: str,llm_api_key: str):
    """
    Detects wallet addresses or ENS domains from a list of posts.
    Converts all items to strings first, then checks for matches.

    Parameters:
    - posts (List): List of posts of any type

    Returns:
    - List[Dict]: List of dicts with 'address' and 'amount' keys
    """

    # Convert everything to strings first
    str_posts = [str(post) for post in posts]
    
    # Then look for matches in all the strings
    eth_pattern = re.compile(r'\b0x[a-fA-F0-9]{40}\b|\b\S+\.eth\b')
    matches = []
    
    for post in str_posts:
        found_matches = eth_pattern.findall(post)
        matches.extend(found_matches)
    
    wallet_balance = get_wallet_balance(private_key, eth_mainnet_rpc_url)
    prompt = get_wallet_decision_prompt(posts, matches, wallet_balance)
    
    response = requests.post(
        url="https://api.hyperbolic.xyz/v1/chat/completions",
        headers={
            "Content-Type": "application/json",
            "Authorization": f"Bearer {llm_api_key}",
        },
        json={
            "messages": [
                {
                    "role": "system",
        	        "content": prompt
                },
                {
                    "role": "user",
                    "content": "Respond only with the wallet address(es) and amount(s) you would like to send to."
                }
            ],
            "model": "meta-llama/Meta-Llama-3.1-70B-Instruct",
            "presence_penalty": 0,
            "temperature": 1,
            "top_p": 0.95,
            "top_k": 40,
        }
    )
    
    if response.status_code == 200:
        print(f"ETH Addresses and amounts chosen from Posts: {response.json()}")
        return response.json()['choices'][0]['message']['content']
    else:
        raise Exception(f"Error generating short-term memory: {response.text}")


-----------------------

/agent/models.py:
-----------------------

from sqlalchemy import Column, Integer, String, Text, DateTime, Boolean, Float, ForeignKey
from sqlalchemy.orm import relationship
from sqlalchemy.sql import func
from sqlalchemy.ext.declarative import declarative_base

Base = declarative_base()


class User(Base):
    __tablename__ = "users"

    id = Column(Integer, primary_key=True, index=True)
    username = Column(String, unique=True, index=True)
    email = Column(String, index=True, default="tee_hee_he@example.com")
    created_at = Column(DateTime(timezone=True), server_default=func.now())
    updated_at = Column(DateTime(timezone=True), onupdate=func.now())

    posts = relationship("Post", back_populates="user")
    comments = relationship("Comment", back_populates="user")
    likes = relationship("Like", back_populates="user")

class Post(Base):
    __tablename__ = "posts"

    id = Column(Integer, primary_key=True, index=True)
    content = Column(Text)
    user_id = Column(Integer, ForeignKey("users.id"))
    username = Column(String)
    created_at = Column(DateTime(timezone=True), server_default=func.now())
    updated_at = Column(DateTime(timezone=True), onupdate=func.now())
    type = Column(String, nullable=False)
    comment_count = Column(Integer, default=0)
    image_path = Column(String)
    tweet_id = Column(String, default=0)

    user = relationship("User", back_populates="posts")
    comments = relationship("Comment", back_populates="post")
    likes = relationship("Like", back_populates="post")

class Comment(Base):
    __tablename__ = "comments"

    id = Column(Integer, primary_key=True, index=True)
    content = Column(String, nullable=False)
    user_id = Column(Integer, ForeignKey("users.id"))
    username = Column(String)
    post_id = Column(Integer, ForeignKey("posts.id"))
    created_at = Column(DateTime(timezone=True), server_default=func.now())
    updated_at = Column(DateTime(timezone=True), onupdate=func.now())
    likes_count = Column(Integer, default=0)

    user = relationship("User", back_populates="comments")
    post = relationship("Post", back_populates="comments")
    likes = relationship("Like", back_populates="comment")

class Like(Base):
    __tablename__ = "likes"

    id = Column(Integer, primary_key=True, index=True)
    user_id = Column(Integer, ForeignKey("users.id"), nullable=False)
    post_id = Column(Integer, ForeignKey("posts.id"), nullable=True)
    comment_id = Column(Integer, ForeignKey("comments.id"), nullable=True)
    is_like = Column(Boolean, nullable=False)  # True for like, False for dislike

    user = relationship("User", back_populates="likes")
    post = relationship("Post", back_populates="likes")
    comment = relationship("Comment", back_populates="likes")

class LongTermMemory(Base):
    __tablename__ = "long_term_memories"

    id = Column(Integer, primary_key=True, index=True)
    content = Column(String, nullable=False)
    embedding = Column(String, nullable=False)  # Store as JSON string
    significance_score = Column(Float, nullable=False)
    created_at = Column(DateTime(timezone=True), server_default=func.now())

# You might want to add a ShortTermMemory model if needed
class ShortTermMemory(Base):
    __tablename__ = "short_term_memories"

    id = Column(Integer, primary_key=True, index=True)
    content = Column(String, nullable=False)
    created_at = Column(DateTime(timezone=True), server_default=func.now())
    # Add any other fields you might need for short-term memory

class TweetPost(Base):
    __tablename__ = "tweet_posts"

    id = Column(Integer, primary_key=True, index=True)
    tweet_id = Column(String, nullable=False)

-----------------------

/agent/pipeline.py:
-----------------------

from dataclasses import dataclass
from typing import List, Tuple, Optional
import json
import os
import time
import re
from random import random
from sqlalchemy.orm import Session

from db.db_setup import get_db
from models import Post, User, TweetPost
from twitter.account import Account

from engines.post_retriever import (
    retrieve_recent_posts,
    fetch_external_context,
    fetch_notification_context,
    format_post_list
)
from engines.short_term_mem import generate_short_term_memory
from engines.long_term_mem import create_embedding, retrieve_relevant_memories, store_memory
from engines.post_maker import generate_post, generate_llm_response
from engines.significance_scorer import score_significance, score_reply_significance
from engines.post_sender import send_post, send_post_API
from engines.wallet_send import transfer_eth, wallet_address_in_post, get_wallet_balance
from engines.follow_user import follow_by_username, decide_to_follow_users

@dataclass
class Config:
    """Configuration for the pipeline."""
    db: Session
    account: Account
    auth: dict
    private_key_hex: str
    eth_mainnet_rpc_url: str
    llm_api_key: str
    openrouter_api_key: str
    openai_api_key: str
    max_reply_rate: float = 1.0  # 100% for testing
    min_posting_significance_score: float = 3.0
    min_storing_memory_significance: float = 7.0
    min_reply_worthiness_score: float = 3.0
    min_follow_score: float = 0.75
    min_eth_balance: float = 0.3
    bot_username: str = "tee_hee_he"
    bot_email: str = "tee_hee_he@example.com"

class PostingPipeline:
    def __init__(self, config: Config):
        self.config = config
        self.ai_user = self._get_or_create_ai_user()

    def _get_or_create_ai_user(self) -> User:
        """Get or create the AI user in the database."""
        ai_user = (self.config.db.query(User)
                  .filter(User.username == self.config.bot_username)
                  .first())
        
        if not ai_user:
            ai_user = User(
                username=self.config.bot_username,
                email=self.config.bot_email
            )
            self.config.db.add(ai_user)
            self.config.db.commit()
        
        return ai_user

    def _handle_wallet_transactions(self, notif_context: List[str]) -> None:
        """Process and execute wallet transactions if conditions are met."""
        balance_ether = get_wallet_balance(
            self.config.private_key_hex,
            self.config.eth_mainnet_rpc_url
        )
        print(f"Agent wallet balance is {balance_ether} ETH now.\n")

        if balance_ether <= self.config.min_eth_balance:
            return

        for _ in range(2):  # Max 2 attempts
            try:
                wallet_data = wallet_address_in_post(
                    notif_context,
                    self.config.private_key_hex,
                    self.config.eth_mainnet_rpc_url,
                    self.config.llm_api_key
                )
                wallets = json.loads(wallet_data)
                
                if not wallets:
                    print("No wallet addresses or amounts to send ETH to.")
                    break

                for wallet in wallets:
                    transfer_eth(
                        self.config.private_key_hex,
                        self.config.eth_mainnet_rpc_url,
                        wallet["address"],
                        wallet["amount"]
                    )
                break
            except (json.JSONDecodeError, KeyError) as e:
                print(f"Error processing wallet data: {e}")
                continue

    def _handle_follows(self, notif_context: List[str]) -> None:
        """Process and execute follow decisions."""
        for _ in range(2):  # Max 2 attempts
            try:
                decision_data = decide_to_follow_users(
                    self.config.db,
                    notif_context,
                    self.config.openrouter_api_key
                )
                decisions = json.loads(decision_data)
                
                if not decisions:
                    print("No users to follow.")
                    break

                for decision in decisions:
                    username = decision["username"]
                    score = decision["score"]
                    
                    if score > self.config.min_follow_score:
                        follow_by_username(self.config.account, username)
                        print(f"user {username} has a high rizz of {score}, now following.")
                    else:
                        print(f"Score {score} for user {username} is too low. Not following.")
                break
            except Exception as e:
                print(f"Error processing follow decisions: {e}")
                continue

    def _should_reply(self, content: str, user_id: str) -> bool:
        """Determine if we should reply to a post."""
        if user_id.lower() == self.config.bot_username:
            return False
        
        if random() > self.config.max_reply_rate:
            return False

        reply_significance_score = score_reply_significance(
            content,
            self.config.llm_api_key
        )
        print(f"Reply significance score: {reply_significance_score}")

        if reply_significance_score >=self.config.min_reply_worthiness_score:
            return True
        else:
            return False

    def _handle_replies(self, external_context: List[Tuple[str, str]]) -> None:
        """Handle replies to mentions and interactions."""
        for content, tweet_id in external_context:
            user_match = re.search(r'@(\w+)', content)
            if not user_match:
                continue

            user_id = user_match.group(1)
            if self._should_reply(content, user_id) == False:
                continue

            try:
                reply_content = generate_post(
                    short_term_memory="",
                    long_term_memories=[],
                    recent_posts=[],
                    external_context=content,
                    llm_api_key=self.config.llm_api_key
                )

                response = self.config.account.reply(reply_content, tweet_id=tweet_id)
                print(f"Replied to {user_id} with: {reply_content}")

                new_reply = Post(
                    content=reply_content,
                    user_id=self.ai_user.id,
                    username=self.ai_user.username,
                    type="reply",
                    tweet_id=response.get('data', {}).get('id')
                )
                self.config.db.add(new_reply)
                self.config.db.commit()

            except Exception as e:
                print(f"Error handling reply: {e}")

    def _post_content(self, content: str) -> Optional[str]:
        """Attempt to post content using available methods."""
        # Try API method first
        tweet_id = send_post_API(self.config.auth, content)
        if tweet_id:
            return tweet_id

        # Fallback to account method
        response = send_post(self.config.account, content)
        return (response.get('data', {})
                .get('create_tweet', {})
                .get('tweet_results', {})
                .get('result', {})
                .get('rest_id'))

    def run(self) -> None:
        """Execute the main pipeline."""
        # Retrieve and format recent posts
        recent_posts = retrieve_recent_posts(self.config.db)
        formatted_posts = format_post_list(recent_posts)
        print(f"Recent posts: {formatted_posts}")

        # Process notifications
        notif_context_tuple = fetch_notification_context(self.config.account)
        existing_tweet_ids = {
            tweet.tweet_id for tweet in 
            self.config.db.query(TweetPost.tweet_id).all()
        }
        
        # Filter new notifications
        filtered_notifs = [
            context for context in notif_context_tuple 
            if context[1] not in existing_tweet_ids
        ]

        # Store processed tweet IDs
        for _, tweet_id in notif_context_tuple:
            self.config.db.add(TweetPost(tweet_id=tweet_id))
        self.config.db.commit()

        notif_context = [context[0] for context in filtered_notifs]
        print("New Notifications:")
        for content, tweet_id in filtered_notifs:
            print(f"- {content}, tweet at https://x.com/user/status/{tweet_id}\n")

        if notif_context:
            self._handle_replies(filtered_notifs)
            time.sleep(5)
            
            self._handle_wallet_transactions(notif_context)
            time.sleep(5)
            
            self._handle_follows(notif_context)
            time.sleep(5)

        # Generate and process memories
        short_term_memory = generate_short_term_memory(
            recent_posts,
            notif_context,
            self.config.llm_api_key
        )
        print(f"Short-term memory: {short_term_memory}")

        short_term_embedding = create_embedding(
            short_term_memory,
            self.config.openai_api_key
        )
        
        long_term_memories = retrieve_relevant_memories(
            self.config.db,
            short_term_embedding
        )
        print(f"Long-term memories: {long_term_memories}")

        # Generate and evaluate new post
        new_post_content = generate_post(
            short_term_memory,
            long_term_memories,
            formatted_posts,
            notif_context,
            self.config.llm_api_key
        ).strip('"')
        print(f"New post content: {new_post_content}")

        significance_score = score_significance(
            new_post_content,
            self.config.llm_api_key
        )
        print(f"Significance score: {significance_score}")

        # Store significant memories
        if significance_score >= self.config.min_storing_memory_significance:
            new_post_embedding = create_embedding(
                new_post_content,
                self.config.openai_api_key
            )
            store_memory(
                self.config.db,
                new_post_content,
                new_post_embedding,
                significance_score
            )

        # Post if significant enough
        if significance_score >= self.config.min_posting_significance_score:
            tweet_id = self._post_content(new_post_content)
            if tweet_id:
                new_post = Post(
                    content=new_post_content,
                    user_id=self.ai_user.id,
                    username=self.ai_user.username,
                    type="text",
                    tweet_id=tweet_id
                )
                self.config.db.add(new_post)
                self.config.db.commit()
                print(f"Posted with tweet_id: {tweet_id}")

-----------------------

/agent/pyproject.toml:
-----------------------

[project]
name = "agent"
version = "0.1.0"
description = "Add your description here"
readme = "README.md"
requires-python = ">=3.11"
dependencies = [
    "eth-keys>=0.6.0",
    "fastapi==0.111.1",
    "numpy>=2.1.2",
    "openai>=1.52.2",
    "pydantic==2.8.2",
    "python-dotenv>=1.0.1",
    "requests==2.31.0",
    "sqlalchemy==2.0.31",
    "tweepy>=4.14.0",
    "twitter-api-client>=0.10.22",
    "uvicorn==0.30.3",
    "web3>=7.4.0",
]


-----------------------

/agent/requirements.txt:
-----------------------

# Python dependencies
fastapi==0.111.1
uvicorn==0.30.3
sqlalchemy==2.0.31
pydantic==2.8.2
requests==2.31.0
openai
python-dotenv
numpy
tweepy
eth_keys
web3
twitter-api-client

-----------------------

/agent/run_pipeline.py:
-----------------------

import os
import time
import random
import json
import secrets
import hashlib
from datetime import datetime, timedelta, time as dt_time
from typing import Tuple, Dict
from pathlib import Path
from requests_oauthlib import OAuth1
from eth_keys import keys
from dotenv import load_dotenv

from db.db_setup import create_database, get_db
from db.db_seed import seed_database
from twitter.account import Account
from engines.post_sender import send_post_API
from pipeline import PostingPipeline, Config

class HumanBehaviorSimulator:
    """Simulates high-volume but natural-looking social media behavior patterns."""
    
    # Extended active hours (24-hour format)
    WEEKDAY_ACTIVE_HOURS = {
        'start': dt_time(6, 0),    # 6 AM
        'peak1': dt_time(9, 0),    # 9 AM
        'peak2': dt_time(12, 0),   # 12 PM
        'peak3': dt_time(15, 0),   # 3 PM
        'peak4': dt_time(19, 0),   # 7 PM
        'peak5': dt_time(22, 0),   # 10 PM
        'end': dt_time(1, 0)       # 1 AM
    }
    
    WEEKEND_ACTIVE_HOURS = {
        'start': dt_time(8, 0),    # 8 AM
        'peak1': dt_time(11, 0),   # 11 AM
        'peak2': dt_time(14, 0),   # 2 PM
        'peak3': dt_time(17, 0),   # 5 PM
        'peak4': dt_time(20, 0),   # 8 PM
        'peak5': dt_time(23, 0),   # 11 PM
        'end': dt_time(2, 0)       # 2 AM
    }
    
    def __init__(self):
        self.last_post_time = None
        self.daily_post_count = 0
        self.max_daily_posts = random.randint(45, 60)  # Target ~50 posts/day
        self.burst_mode = False
        self.burst_count = 0
        self.max_burst = random.randint(3, 5)
        self.last_burst_time = None
        
    def is_active_hour(self) -> bool:
        """Determine if current time is within active hours."""
        current_time = datetime.now().time()
        current_day = datetime.now().weekday()
        hours = self.WEEKEND_ACTIVE_HOURS if current_day >= 5 else self.WEEKDAY_ACTIVE_HOURS
        
        # Handle day transition
        if hours['end'] < hours['start']:
            return current_time >= hours['start'] or current_time <= hours['end']
        return hours['start'] <= current_time <= hours['end']
    
    def get_post_probability(self) -> float:
        """Calculate probability of posting based on time and previous activity."""
        if not self.is_active_hour():
            return 0.2  # Still maintain some off-hours activity
        
        current_time = datetime.now().time()
        current_day = datetime.now().weekday()
        hours = self.WEEKEND_ACTIVE_HOURS if current_day >= 5 else self.WEEKDAY_ACTIVE_HOURS
        
        # Base probability higher to achieve volume
        prob = 0.7
        
        # Check if we're in burst mode
        if self.burst_mode:
            if self.burst_count >= self.max_burst:
                self.burst_mode = False
                self.burst_count = 0
                prob *= 0.3  # Cool down after burst
            else:
                prob = 0.9  # High probability during burst
                
        # Start new burst randomly
        elif (not self.last_burst_time or 
              (datetime.now() - self.last_burst_time).total_seconds() > 1800):  # 30 min
            if random.random() < 0.2:  # 20% chance to start burst
                self.burst_mode = True
                self.last_burst_time = datetime.now()
                prob = 0.9
        
        # Increase probability during peak hours
        for peak_hour in ['peak1', 'peak2', 'peak3', 'peak4', 'peak5']:
            peak_time = hours[peak_hour]
            time_diff = abs(current_time.hour - peak_time.hour)
            if time_diff <= 1:
                prob *= 1.3
                break
                
        # Minimum gap between regular posts (2-5 minutes)
        if self.last_post_time:
            minutes_since_last = (datetime.now() - self.last_post_time).total_seconds() / 60
            if minutes_since_last < 2:
                return 0
            elif minutes_since_last < 5:
                prob *= 0.5
                
        # Adjust for daily target
        hours_remaining = (24 - datetime.now().hour)
        target_remaining = self.max_daily_posts - self.daily_post_count
        if hours_remaining > 0:
            current_rate = target_remaining / hours_remaining
            if current_rate > 3:  # Need to post more frequently
                prob *= 1.2
            elif current_rate < 1:  # Can slow down
                prob *= 0.8
                
        return min(prob, 1.0)
    
    def should_post(self) -> bool:
        """Decide whether to post based on various factors."""
        # Reset daily count if it's a new day
        if self.last_post_time and self.last_post_time.date() != datetime.now().date():
            self.daily_post_count = 0
            self.max_daily_posts = random.randint(45, 60)
            self.burst_mode = False
            self.burst_count = 0
        
        prob = self.get_post_probability()
        should_post = random.random() < prob
        
        if should_post:
            self.last_post_time = datetime.now()
            self.daily_post_count += 1
            if self.burst_mode:
                self.burst_count += 1
            
        return should_post


class PipelineRunner:
    def __init__(self):
        self.setup_environment()
        self.db = next(get_db())
        self.config = self.create_config()
        self.pipeline = PostingPipeline(self.config)
        self.behavior_simulator = HumanBehaviorSimulator()  # Initialize the simulator

    def setup_environment(self) -> None:
        """Initialize environment and database."""
        load_dotenv()
        
        db_path = Path("./data/agents.db")
        if not db_path.exists():
            print("Creating database...")
            db_path.parent.mkdir(parents=True, exist_ok=True)
            create_database()
            print("Seeding database...")
            seed_database()
        else:
            print("Database already exists. Skipping creation and seeding.")

    def generate_eth_account(self) -> Tuple[str, str]:
        """Generate a new Ethereum account with private key and address."""
        random_seed = secrets.token_bytes(32)
        hashed_output = hashlib.sha256(random_seed).digest()
        private_key = keys.PrivateKey(hashed_output)
        private_key_hex = private_key.to_hex()
        eth_address = private_key.public_key.to_checksum_address()
        return private_key_hex, eth_address

    def get_api_keys(self) -> Dict[str, str]:
        """Retrieve API keys from environment variables."""
        return {
            "llm_api_key": os.getenv("HYPERBOLIC_API_KEY"),
            "openai_api_key": os.getenv("OPENAI_API_KEY"),
            "openrouter_api_key": os.getenv("OPENROUTER_API_KEY"),
        }

    def get_twitter_config(self) -> Tuple[OAuth1, Account]:
        """Set up Twitter authentication and account."""
        auth = OAuth1(
            os.getenv("X_CONSUMER_KEY"),
            os.getenv("X_CONSUMER_SECRET"),
            os.getenv("X_ACCESS_TOKEN"),
            os.getenv("X_ACCESS_TOKEN_SECRET")
        )
        
        auth_tokens = json.loads(os.getenv("X_AUTH_TOKENS"))
        account = Account(cookies=auth_tokens)
        
        return auth, account

    def create_config(self) -> Config:
        """Create pipeline configuration."""
        api_keys = self.get_api_keys()
        auth, account = self.get_twitter_config()
        private_key_hex, eth_address = self.generate_eth_account()
        
        print(f"Generated agent exclusively-owned wallet: {eth_address}")
        tweet_id = send_post_API(auth, f'My wallet is {eth_address}')
        print(f"Wallet announcement tweet: https://x.com/user/status/{tweet_id}")
        
        return Config(
            db=self.db,
            account=account,
            auth=auth,
            private_key_hex=private_key_hex,
            eth_mainnet_rpc_url=os.getenv("ETH_MAINNET_RPC_URL"),
            **api_keys
        )

    def get_timing_parameters(self) -> Tuple[datetime, timedelta]:
        """Calculate next activation time and duration."""
        if self.behavior_simulator.burst_mode:
            # Shorter cycles during burst mode
            delay_minutes = random.uniform(1, 3)
            duration_minutes = random.uniform(5, 10)
        else:
            # Regular timing
            if not self.behavior_simulator.is_active_hour():
                delay_minutes = random.uniform(10, 20)
                duration_minutes = random.uniform(5, 10)
            else:
                delay_minutes = random.uniform(3, 8)
                duration_minutes = random.uniform(8, 15)
        
        activation_time = datetime.now() + timedelta(minutes=delay_minutes)
        active_duration = timedelta(minutes=duration_minutes)
        return activation_time, active_duration

    def get_next_run_time(self) -> datetime:
        """Calculate next run time with variable delays."""
        if self.behavior_simulator.burst_mode:
            # Quick checks during bursts
            delay_seconds = random.uniform(30, 90)
        else:
            # Regular timing
            if self.behavior_simulator.is_active_hour():
                delay_seconds = random.uniform(60, 180)  # 1-3 minutes
            else:
                delay_seconds = random.uniform(180, 300)  # 3-5 minutes
                
        return datetime.now() + timedelta(seconds=delay_seconds)

    def run_pipeline_cycle(self) -> None:
        """Run a single pipeline cycle."""
        activation_time, active_duration = self.get_timing_parameters()
        deactivation_time = activation_time + active_duration

        print(f"\nNext cycle:")
        print(f"Activation time: {activation_time.strftime('%I:%M:%S %p')}")
        print(f"Deactivation time: {deactivation_time.strftime('%I:%M:%S %p')}")
        print(f"Duration: {active_duration.total_seconds() / 60:.1f} minutes")
        print(f"Daily posts so far: {self.behavior_simulator.daily_post_count}")
        print(f"Burst mode: {'Yes' if self.behavior_simulator.burst_mode else 'No'}")

        # Wait for activation time
        while datetime.now() < activation_time:
            time.sleep(60)

        print(f"\nPipeline activated at: {datetime.now().strftime('%H:%M:%S')}")
        next_run = self.get_next_run_time()

        while datetime.now() < deactivation_time:
            if datetime.now() >= next_run:
                if self.behavior_simulator.should_post():
                    print(f"Running pipeline at: {datetime.now().strftime('%H:%M:%S')}")
                    try:
                        self.pipeline.run()
                    except Exception as e:
                        print(f"Error running pipeline: {e}")
                else:
                    print("Skipping post based on behavior pattern...")

                next_run = self.get_next_run_time()
                print(
                    f"Next run scheduled for: {next_run.strftime('%H:%M:%S')} "
                    f"({(next_run - datetime.now()).total_seconds():.1f} seconds from now)"
                )

            time.sleep(1)

        print(f"Pipeline deactivated at: {datetime.now().strftime('%H:%M:%S')}")

    def run(self) -> None:
        """Main execution loop."""
        print("\nPerforming initial pipeline run...")
        try:
            self.pipeline.run()
            print("Initial run completed successfully.")
        except Exception as e:
            print(f"Error during initial run: {e}")

        print("Starting continuous pipeline process...")
        while True:
            try:
                self.run_pipeline_cycle()
            except Exception as e:
                print(f"Error in pipeline cycle: {e}")
                continue

def main():
    try:
        runner = PipelineRunner()
        runner.run()
    except KeyboardInterrupt:
        print("\nProcess terminated by user")

if __name__ == "__main__":
    main()

-----------------------

/agent/signin.py:
-----------------------

from twitter.account import Account
import json
import os
import dotenv

dotenv.load_dotenv()

cookies = os.environ.get("X_AUTH_TOKENS")
auth_tokens = json.loads(cookies)

account = Account(cookies=auth_tokens)
timeline = account.home_latest_timeline(10)
print(timeline)


-----------------------

/agent/uv.lock:
-----------------------

version = 1
requires-python = ">=3.11"
resolution-markers = [
    "python_full_version < '3.13'",
    "python_full_version >= '3.13'",
]

[[package]]
name = "agent"
version = "0.1.0"
source = { virtual = "." }
dependencies = [
    { name = "eth-keys" },
    { name = "fastapi" },
    { name = "numpy" },
    { name = "openai" },
    { name = "pydantic" },
    { name = "python-dotenv" },
    { name = "requests" },
    { name = "sqlalchemy" },
    { name = "tweepy" },
    { name = "twitter-api-client" },
    { name = "uvicorn" },
    { name = "web3" },
]

[package.metadata]
requires-dist = [
    { name = "eth-keys", specifier = ">=0.6.0" },
    { name = "fastapi", specifier = "==0.111.1" },
    { name = "numpy", specifier = ">=2.1.2" },
    { name = "openai", specifier = ">=1.52.2" },
    { name = "pydantic", specifier = "==2.8.2" },
    { name = "python-dotenv", specifier = ">=1.0.1" },
    { name = "requests", specifier = "==2.31.0" },
    { name = "sqlalchemy", specifier = "==2.0.31" },
    { name = "tweepy", specifier = ">=4.14.0" },
    { name = "twitter-api-client", specifier = ">=0.10.22" },
    { name = "uvicorn", specifier = "==0.30.3" },
    { name = "web3", specifier = ">=7.4.0" },
]

[[package]]
name = "aiofiles"
version = "24.1.0"
source = { registry = "https://pypi.org/simple" }
sdist = { url = "https://files.pythonhosted.org/packages/0b/03/a88171e277e8caa88a4c77808c20ebb04ba74cc4681bf1e9416c862de237/aiofiles-24.1.0.tar.gz", hash = "sha256:22a075c9e5a3810f0c2e48f3008c94d68c65d763b9b03857924c99e57355166c", size = 30247 }
wheels = [
    { url = "https://files.pythonhosted.org/packages/a5/45/30bb92d442636f570cb5651bc661f52b610e2eec3f891a5dc3a4c3667db0/aiofiles-24.1.0-py3-none-any.whl", hash = "sha256:b4ec55f4195e3eb5d7abd1bf7e061763e864dd4954231fb8539a0ef8bb8260e5", size = 15896 },
]

[[package]]
name = "aiohappyeyeballs"
version = "2.4.3"
source = { registry = "https://pypi.org/simple" }
sdist = { url = "https://files.pythonhosted.org/packages/bc/69/2f6d5a019bd02e920a3417689a89887b39ad1e350b562f9955693d900c40/aiohappyeyeballs-2.4.3.tar.gz", hash = "sha256:75cf88a15106a5002a8eb1dab212525c00d1f4c0fa96e551c9fbe6f09a621586", size = 21809 }
wheels = [
    { url = "https://files.pythonhosted.org/packages/f7/d8/120cd0fe3e8530df0539e71ba9683eade12cae103dd7543e50d15f737917/aiohappyeyeballs-2.4.3-py3-none-any.whl", hash = "sha256:8a7a83727b2756f394ab2895ea0765a0a8c475e3c71e98d43d76f22b4b435572", size = 14742 },
]

[[package]]
name = "aiohttp"
version = "3.10.10"
source = { registry = "https://pypi.org/simple" }
dependencies = [
    { name = "aiohappyeyeballs" },
    { name = "aiosignal" },
    { name = "attrs" },
    { name = "frozenlist" },
    { name = "multidict" },
    { name = "yarl" },
]
sdist = { url = "https://files.pythonhosted.org/packages/17/7e/16e57e6cf20eb62481a2f9ce8674328407187950ccc602ad07c685279141/aiohttp-3.10.10.tar.gz", hash = "sha256:0631dd7c9f0822cc61c88586ca76d5b5ada26538097d0f1df510b082bad3411a", size = 7542993 }
wheels = [
    { url = "https://files.pythonhosted.org/packages/72/31/3c351d17596194e5a38ef169a4da76458952b2497b4b54645b9d483cbbb0/aiohttp-3.10.10-cp311-cp311-macosx_10_9_universal2.whl", hash = "sha256:c30a0eafc89d28e7f959281b58198a9fa5e99405f716c0289b7892ca345fe45f", size = 586501 },
    { url = "https://files.pythonhosted.org/packages/a4/a8/a559d09eb08478cdead6b7ce05b0c4a133ba27fcdfa91e05d2e62867300d/aiohttp-3.10.10-cp311-cp311-macosx_10_9_x86_64.whl", hash = "sha256:258c5dd01afc10015866114e210fb7365f0d02d9d059c3c3415382ab633fcbcb", size = 398993 },
    { url = "https://files.pythonhosted.org/packages/c5/47/7736d4174613feef61d25332c3bd1a4f8ff5591fbd7331988238a7299485/aiohttp-3.10.10-cp311-cp311-macosx_11_0_arm64.whl", hash = "sha256:15ecd889a709b0080f02721255b3f80bb261c2293d3c748151274dfea93ac871", size = 390647 },
    { url = "https://files.pythonhosted.org/packages/27/21/e9ba192a04b7160f5a8952c98a1de7cf8072ad150fa3abd454ead1ab1d7f/aiohttp-3.10.10-cp311-cp311-manylinux_2_17_aarch64.manylinux2014_aarch64.whl", hash = "sha256:f3935f82f6f4a3820270842e90456ebad3af15810cf65932bd24da4463bc0a4c", size = 1306481 },
    { url = "https://files.pythonhosted.org/packages/cf/50/f364c01c8d0def1dc34747b2470969e216f5a37c7ece00fe558810f37013/aiohttp-3.10.10-cp311-cp311-manylinux_2_17_ppc64le.manylinux2014_ppc64le.whl", hash = "sha256:413251f6fcf552a33c981c4709a6bba37b12710982fec8e558ae944bfb2abd38", size = 1344652 },
    { url = "https://files.pythonhosted.org/packages/1d/c2/74f608e984e9b585649e2e83883facad6fa3fc1d021de87b20cc67e8e5ae/aiohttp-3.10.10-cp311-cp311-manylinux_2_17_s390x.manylinux2014_s390x.whl", hash = "sha256:d1720b4f14c78a3089562b8875b53e36b51c97c51adc53325a69b79b4b48ebcb", size = 1378498 },
    { url = "https://files.pythonhosted.org/packages/9f/a7/05a48c7c0a7a80a5591b1203bf1b64ca2ed6a2050af918d09c05852dc42b/aiohttp-3.10.10-cp311-cp311-manylinux_2_17_x86_64.manylinux2014_x86_64.whl", hash = "sha256:679abe5d3858b33c2cf74faec299fda60ea9de62916e8b67e625d65bf069a3b7", size = 1292718 },
    { url = "https://files.pythonhosted.org/packages/7d/78/a925655018747e9790350180330032e27d6e0d7ed30bde545fae42f8c49c/aiohttp-3.10.10-cp311-cp311-manylinux_2_5_i686.manylinux1_i686.manylinux_2_17_i686.manylinux2014_i686.whl", hash = "sha256:79019094f87c9fb44f8d769e41dbb664d6e8fcfd62f665ccce36762deaa0e911", size = 1251776 },
    { url = "https://files.pythonhosted.org/packages/47/9d/85c6b69f702351d1236594745a4fdc042fc43f494c247a98dac17e004026/aiohttp-3.10.10-cp311-cp311-musllinux_1_2_aarch64.whl", hash = "sha256:fe2fb38c2ed905a2582948e2de560675e9dfbee94c6d5ccdb1301c6d0a5bf092", size = 1271716 },
    { url = "https://files.pythonhosted.org/packages/7f/a7/55fc805ff9b14af818903882ece08e2235b12b73b867b521b92994c52b14/aiohttp-3.10.10-cp311-cp311-musllinux_1_2_i686.whl", hash = "sha256:a3f00003de6eba42d6e94fabb4125600d6e484846dbf90ea8e48a800430cc142", size = 1266263 },
    { url = "https://files.pythonhosted.org/packages/1f/ec/d2be2ca7b063e4f91519d550dbc9c1cb43040174a322470deed90b3d3333/aiohttp-3.10.10-cp311-cp311-musllinux_1_2_ppc64le.whl", hash = "sha256:1bbb122c557a16fafc10354b9d99ebf2f2808a660d78202f10ba9d50786384b9", size = 1321617 },
    { url = "https://files.pythonhosted.org/packages/c9/a3/b29f7920e1cd0a9a68a45dd3eb16140074d2efb1518d2e1f3e140357dc37/aiohttp-3.10.10-cp311-cp311-musllinux_1_2_s390x.whl", hash = "sha256:30ca7c3b94708a9d7ae76ff281b2f47d8eaf2579cd05971b5dc681db8caac6e1", size = 1339227 },
    { url = "https://files.pythonhosted.org/packages/8a/81/34b67235c47e232d807b4bbc42ba9b927c7ce9476872372fddcfd1e41b3d/aiohttp-3.10.10-cp311-cp311-musllinux_1_2_x86_64.whl", hash = "sha256:df9270660711670e68803107d55c2b5949c2e0f2e4896da176e1ecfc068b974a", size = 1299068 },
    { url = "https://files.pythonhosted.org/packages/04/1f/26a7fe11b6ad3184f214733428353c89ae9fe3e4f605a657f5245c5e720c/aiohttp-3.10.10-cp311-cp311-win32.whl", hash = "sha256:aafc8ee9b742ce75044ae9a4d3e60e3d918d15a4c2e08a6c3c3e38fa59b92d94", size = 362223 },
    { url = "https://files.pythonhosted.org/packages/10/91/85dcd93f64011434359ce2666bece981f08d31bc49df33261e625b28595d/aiohttp-3.10.10-cp311-cp311-win_amd64.whl", hash = "sha256:362f641f9071e5f3ee6f8e7d37d5ed0d95aae656adf4ef578313ee585b585959", size = 381576 },
    { url = "https://files.pythonhosted.org/packages/ae/99/4c5aefe5ad06a1baf206aed6598c7cdcbc7c044c46801cd0d1ecb758cae3/aiohttp-3.10.10-cp312-cp312-macosx_10_9_universal2.whl", hash = "sha256:9294bbb581f92770e6ed5c19559e1e99255e4ca604a22c5c6397b2f9dd3ee42c", size = 583536 },
    { url = "https://files.pythonhosted.org/packages/a9/36/8b3bc49b49cb6d2da40ee61ff15dbcc44fd345a3e6ab5bb20844df929821/aiohttp-3.10.10-cp312-cp312-macosx_10_9_x86_64.whl", hash = "sha256:a8fa23fe62c436ccf23ff930149c047f060c7126eae3ccea005f0483f27b2e28", size = 395693 },
    { url = "https://files.pythonhosted.org/packages/e1/77/0aa8660dcf11fa65d61712dbb458c4989de220a844bd69778dff25f2d50b/aiohttp-3.10.10-cp312-cp312-macosx_11_0_arm64.whl", hash = "sha256:5c6a5b8c7926ba5d8545c7dd22961a107526562da31a7a32fa2456baf040939f", size = 390898 },
    { url = "https://files.pythonhosted.org/packages/38/d2/b833d95deb48c75db85bf6646de0a697e7fb5d87bd27cbade4f9746b48b1/aiohttp-3.10.10-cp312-cp312-manylinux_2_17_aarch64.manylinux2014_aarch64.whl", hash = "sha256:007ec22fbc573e5eb2fb7dec4198ef8f6bf2fe4ce20020798b2eb5d0abda6138", size = 1312060 },
    { url = "https://files.pythonhosted.org/packages/aa/5f/29fd5113165a0893de8efedf9b4737e0ba92dfcd791415a528f947d10299/aiohttp-3.10.10-cp312-cp312-manylinux_2_17_ppc64le.manylinux2014_ppc64le.whl", hash = "sha256:9627cc1a10c8c409b5822a92d57a77f383b554463d1884008e051c32ab1b3742", size = 1350553 },
    { url = "https://files.pythonhosted.org/packages/ad/cc/f835f74b7d344428469200105236d44606cfa448be1e7c95ca52880d9bac/aiohttp-3.10.10-cp312-cp312-manylinux_2_17_s390x.manylinux2014_s390x.whl", hash = "sha256:50edbcad60d8f0e3eccc68da67f37268b5144ecc34d59f27a02f9611c1d4eec7", size = 1392646 },
    { url = "https://files.pythonhosted.org/packages/bf/fe/1332409d845ca601893bbf2d76935e0b93d41686e5f333841c7d7a4a770d/aiohttp-3.10.10-cp312-cp312-manylinux_2_17_x86_64.manylinux2014_x86_64.whl", hash = "sha256:a45d85cf20b5e0d0aa5a8dca27cce8eddef3292bc29d72dcad1641f4ed50aa16", size = 1306310 },
    { url = "https://files.pythonhosted.org/packages/e4/a1/25a7633a5a513278a9892e333501e2e69c83e50be4b57a62285fb7a008c3/aiohttp-3.10.10-cp312-cp312-manylinux_2_5_i686.manylinux1_i686.manylinux_2_17_i686.manylinux2014_i686.whl", hash = "sha256:0b00807e2605f16e1e198f33a53ce3c4523114059b0c09c337209ae55e3823a8", size = 1260255 },
    { url = "https://files.pythonhosted.org/packages/f2/39/30eafe89e0e2a06c25e4762844c8214c0c0cd0fd9ffc3471694a7986f421/aiohttp-3.10.10-cp312-cp312-musllinux_1_2_aarch64.whl", hash = "sha256:f2d4324a98062be0525d16f768a03e0bbb3b9fe301ceee99611dc9a7953124e6", size = 1271141 },
    { url = "https://files.pythonhosted.org/packages/5b/fc/33125df728b48391ef1fcb512dfb02072158cc10d041414fb79803463020/aiohttp-3.10.10-cp312-cp312-musllinux_1_2_i686.whl", hash = "sha256:438cd072f75bb6612f2aca29f8bd7cdf6e35e8f160bc312e49fbecab77c99e3a", size = 1280244 },
    { url = "https://files.pythonhosted.org/packages/3b/61/e42bf2c2934b5caa4e2ec0b5e5fd86989adb022b5ee60c2572a9d77cf6fe/aiohttp-3.10.10-cp312-cp312-musllinux_1_2_ppc64le.whl", hash = "sha256:baa42524a82f75303f714108fea528ccacf0386af429b69fff141ffef1c534f9", size = 1316805 },
    { url = "https://files.pythonhosted.org/packages/18/32/f52a5e2ae9ad3bba10e026a63a7a23abfa37c7d97aeeb9004eaa98df3ce3/aiohttp-3.10.10-cp312-cp312-musllinux_1_2_s390x.whl", hash = "sha256:a7d8d14fe962153fc681f6366bdec33d4356f98a3e3567782aac1b6e0e40109a", size = 1343930 },
    { url = "https://files.pythonhosted.org/packages/05/be/6a403b464dcab3631fe8e27b0f1d906d9e45c5e92aca97ee007e5a895560/aiohttp-3.10.10-cp312-cp312-musllinux_1_2_x86_64.whl", hash = "sha256:c1277cd707c465cd09572a774559a3cc7c7a28802eb3a2a9472588f062097205", size = 1306186 },
    { url = "https://files.pythonhosted.org/packages/8e/fd/bb50fe781068a736a02bf5c7ad5f3ab53e39f1d1e63110da6d30f7605edc/aiohttp-3.10.10-cp312-cp312-win32.whl", hash = "sha256:59bb3c54aa420521dc4ce3cc2c3fe2ad82adf7b09403fa1f48ae45c0cbde6628", size = 359289 },
    { url = "https://files.pythonhosted.org/packages/70/9e/5add7e240f77ef67c275c82cc1d08afbca57b77593118c1f6e920ae8ad3f/aiohttp-3.10.10-cp312-cp312-win_amd64.whl", hash = "sha256:0e1b370d8007c4ae31ee6db7f9a2fe801a42b146cec80a86766e7ad5c4a259cf", size = 379313 },
    { url = "https://files.pythonhosted.org/packages/b1/eb/618b1b76c7fe8082a71c9d62e3fe84c5b9af6703078caa9ec57850a12080/aiohttp-3.10.10-cp313-cp313-macosx_10_13_universal2.whl", hash = "sha256:ad7593bb24b2ab09e65e8a1d385606f0f47c65b5a2ae6c551db67d6653e78c28", size = 576114 },
    { url = "https://files.pythonhosted.org/packages/aa/37/3126995d7869f8b30d05381b81a2d4fb4ec6ad313db788e009bc6d39c211/aiohttp-3.10.10-cp313-cp313-macosx_10_13_x86_64.whl", hash = "sha256:1eb89d3d29adaf533588f209768a9c02e44e4baf832b08118749c5fad191781d", size = 391901 },
    { url = "https://files.pythonhosted.org/packages/3e/f2/8fdfc845be1f811c31ceb797968523813f8e1263ee3e9120d61253f6848f/aiohttp-3.10.10-cp313-cp313-macosx_11_0_arm64.whl", hash = "sha256:3fe407bf93533a6fa82dece0e74dbcaaf5d684e5a51862887f9eaebe6372cd79", size = 387418 },
    { url = "https://files.pythonhosted.org/packages/60/d5/33d2061d36bf07e80286e04b7e0a4de37ce04b5ebfed72dba67659a05250/aiohttp-3.10.10-cp313-cp313-manylinux_2_17_aarch64.manylinux2014_aarch64.whl", hash = "sha256:50aed5155f819873d23520919e16703fc8925e509abbb1a1491b0087d1cd969e", size = 1287073 },
    { url = "https://files.pythonhosted.org/packages/00/52/affb55be16a4747740bd630b4c002dac6c5eac42f9bb64202fc3cf3f1930/aiohttp-3.10.10-cp313-cp313-manylinux_2_17_ppc64le.manylinux2014_ppc64le.whl", hash = "sha256:4f05e9727ce409358baa615dbeb9b969db94324a79b5a5cea45d39bdb01d82e6", size = 1323612 },
    { url = "https://files.pythonhosted.org/packages/94/f2/cddb69b975387daa2182a8442566971d6410b8a0179bb4540d81c97b1611/aiohttp-3.10.10-cp313-cp313-manylinux_2_17_s390x.manylinux2014_s390x.whl", hash = "sha256:3dffb610a30d643983aeb185ce134f97f290f8935f0abccdd32c77bed9388b42", size = 1368406 },
    { url = "https://files.pythonhosted.org/packages/c1/e4/afba7327da4d932da8c6e29aecaf855f9d52dace53ac15bfc8030a246f1b/aiohttp-3.10.10-cp313-cp313-manylinux_2_17_x86_64.manylinux2014_x86_64.whl", hash = "sha256:aa6658732517ddabe22c9036479eabce6036655ba87a0224c612e1ae6af2087e", size = 1282761 },
    { url = "https://files.pythonhosted.org/packages/9f/6b/364856faa0c9031ea76e24ef0f7fef79cddd9fa8e7dba9a1771c6acc56b5/aiohttp-3.10.10-cp313-cp313-manylinux_2_5_i686.manylinux1_i686.manylinux_2_17_i686.manylinux2014_i686.whl", hash = "sha256:741a46d58677d8c733175d7e5aa618d277cd9d880301a380fd296975a9cdd7bc", size = 1236518 },
    { url = "https://files.pythonhosted.org/packages/46/af/c382846f8356fe64a7b5908bb9b477457aa23b71be7ed551013b7b7d4d87/aiohttp-3.10.10-cp313-cp313-musllinux_1_2_aarch64.whl", hash = "sha256:e00e3505cd80440f6c98c6d69269dcc2a119f86ad0a9fd70bccc59504bebd68a", size = 1250344 },
    { url = "https://files.pythonhosted.org/packages/87/53/294f87fc086fd0772d0ab82497beb9df67f0f27a8b3dd5742a2656db2bc6/aiohttp-3.10.10-cp313-cp313-musllinux_1_2_i686.whl", hash = "sha256:ffe595f10566f8276b76dc3a11ae4bb7eba1aac8ddd75811736a15b0d5311414", size = 1248956 },
    { url = "https://files.pythonhosted.org/packages/86/30/7d746717fe11bdfefb88bb6c09c5fc985d85c4632da8bb6018e273899254/aiohttp-3.10.10-cp313-cp313-musllinux_1_2_ppc64le.whl", hash = "sha256:bdfcf6443637c148c4e1a20c48c566aa694fa5e288d34b20fcdc58507882fed3", size = 1293379 },
    { url = "https://files.pythonhosted.org/packages/48/b9/45d670a834458db67a24258e9139ba61fa3bd7d69b98ecf3650c22806f8f/aiohttp-3.10.10-cp313-cp313-musllinux_1_2_s390x.whl", hash = "sha256:d183cf9c797a5291e8301790ed6d053480ed94070637bfaad914dd38b0981f67", size = 1320108 },
    { url = "https://files.pythonhosted.org/packages/72/8c/804bb2e837a175635d2000a0659eafc15b2e9d92d3d81c8f69e141ecd0b0/aiohttp-3.10.10-cp313-cp313-musllinux_1_2_x86_64.whl", hash = "sha256:77abf6665ae54000b98b3c742bc6ea1d1fb31c394bcabf8b5d2c1ac3ebfe7f3b", size = 1281546 },
    { url = "https://files.pythonhosted.org/packages/89/c0/862e6a9de3d6eeb126cd9d9ea388243b70df9b871ce1a42b193b7a4a77fc/aiohttp-3.10.10-cp313-cp313-win32.whl", hash = "sha256:4470c73c12cd9109db8277287d11f9dd98f77fc54155fc71a7738a83ffcc8ea8", size = 357516 },
    { url = "https://files.pythonhosted.org/packages/ae/63/3e1aee3e554263f3f1011cca50d78a4894ae16ce99bf78101ac3a2f0ef74/aiohttp-3.10.10-cp313-cp313-win_amd64.whl", hash = "sha256:486f7aabfa292719a2753c016cc3a8f8172965cabb3ea2e7f7436c7f5a22a151", size = 376785 },
]

[[package]]
name = "aiosignal"
version = "1.3.1"
source = { registry = "https://pypi.org/simple" }
dependencies = [
    { name = "frozenlist" },
]
sdist = { url = "https://files.pythonhosted.org/packages/ae/67/0952ed97a9793b4958e5736f6d2b346b414a2cd63e82d05940032f45b32f/aiosignal-1.3.1.tar.gz", hash = "sha256:54cd96e15e1649b75d6c87526a6ff0b6c1b0dd3459f43d9ca11d48c339b68cfc", size = 19422 }
wheels = [
    { url = "https://files.pythonhosted.org/packages/76/ac/a7305707cb852b7e16ff80eaf5692309bde30e2b1100a1fcacdc8f731d97/aiosignal-1.3.1-py3-none-any.whl", hash = "sha256:f8376fb07dd1e86a584e4fcdec80b36b7f81aac666ebc724e2c090300dd83b17", size = 7617 },
]

[[package]]
name = "annotated-types"
version = "0.7.0"
source = { registry = "https://pypi.org/simple" }
sdist = { url = "https://files.pythonhosted.org/packages/ee/67/531ea369ba64dcff5ec9c3402f9f51bf748cec26dde048a2f973a4eea7f5/annotated_types-0.7.0.tar.gz", hash = "sha256:aff07c09a53a08bc8cfccb9c85b05f1aa9a2a6f23728d790723543408344ce89", size = 16081 }
wheels = [
    { url = "https://files.pythonhosted.org/packages/78/b6/6307fbef88d9b5ee7421e68d78a9f162e0da4900bc5f5793f6d3d0e34fb8/annotated_types-0.7.0-py3-none-any.whl", hash = "sha256:1f02e8b43a8fbbc3f3e0d4f0f4bfc8131bcb4eebe8849b8e5c773f3a1c582a53", size = 13643 },
]

[[package]]
name = "anyio"
version = "4.6.2.post1"
source = { registry = "https://pypi.org/simple" }
dependencies = [
    { name = "idna" },
    { name = "sniffio" },
]
sdist = { url = "https://files.pythonhosted.org/packages/9f/09/45b9b7a6d4e45c6bcb5bf61d19e3ab87df68e0601fa8c5293de3542546cc/anyio-4.6.2.post1.tar.gz", hash = "sha256:4c8bc31ccdb51c7f7bd251f51c609e038d63e34219b44aa86e47576389880b4c", size = 173422 }
wheels = [
    { url = "https://files.pythonhosted.org/packages/e4/f5/f2b75d2fc6f1a260f340f0e7c6a060f4dd2961cc16884ed851b0d18da06a/anyio-4.6.2.post1-py3-none-any.whl", hash = "sha256:6d170c36fba3bdd840c73d3868c1e777e33676a69c3a72cf0a0d5d6d8009b61d", size = 90377 },
]

[[package]]
name = "attrs"
version = "24.2.0"
source = { registry = "https://pypi.org/simple" }
sdist = { url = "https://files.pythonhosted.org/packages/fc/0f/aafca9af9315aee06a89ffde799a10a582fe8de76c563ee80bbcdc08b3fb/attrs-24.2.0.tar.gz", hash = "sha256:5cfb1b9148b5b086569baec03f20d7b6bf3bcacc9a42bebf87ffaaca362f6346", size = 792678 }
wheels = [
    { url = "https://files.pythonhosted.org/packages/6a/21/5b6702a7f963e95456c0de2d495f67bf5fd62840ac655dc451586d23d39a/attrs-24.2.0-py3-none-any.whl", hash = "sha256:81921eb96de3191c8258c199618104dd27ac608d9366f5e35d011eae1867ede2", size = 63001 },
]

[[package]]
name = "bitarray"
version = "3.0.0"
source = { registry = "https://pypi.org/simple" }
sdist = { url = "https://files.pythonhosted.org/packages/85/62/dcfac53d22ef7e904ed10a8e710a36391d2d6753c34c869b51bfc5e4ad54/bitarray-3.0.0.tar.gz", hash = "sha256:a2083dc20f0d828a7cdf7a16b20dae56aab0f43dc4f347a3b3039f6577992b03", size = 126627 }
wheels = [
    { url = "https://files.pythonhosted.org/packages/61/41/321edc0fbf7e8c88552d5ff9ee07777d58e2078f2706c6478bc6651b1945/bitarray-3.0.0-cp311-cp311-macosx_10_9_universal2.whl", hash = "sha256:44c3e78b60070389b824d5a654afa1c893df723153c81904088d4922c3cfb6ac", size = 172452 },
    { url = "https://files.pythonhosted.org/packages/48/92/4c312d6d55ac30dae96749830c9f5007a914efcb591ee0828914078eec9f/bitarray-3.0.0-cp311-cp311-macosx_10_9_x86_64.whl", hash = "sha256:545d36332de81e4742a845a80df89530ff193213a50b4cbef937ed5a44c0e5e5", size = 123502 },
    { url = "https://files.pythonhosted.org/packages/75/2c/9f3ed70ffac8e6d2b0880e132d9e5024e4ef9404a24220deca8dbd702f15/bitarray-3.0.0-cp311-cp311-macosx_11_0_arm64.whl", hash = "sha256:8a9eb510cde3fa78c2e302bece510bf5ed494ec40e6b082dec753d6e22d5d1b1", size = 121363 },
    { url = "https://files.pythonhosted.org/packages/48/e0/8ec59416aaa7ca1461a0268c0fe2fbdc8d574ac41e307980f555b773d5f6/bitarray-3.0.0-cp311-cp311-manylinux_2_17_aarch64.manylinux2014_aarch64.whl", hash = "sha256:9e3727ab63dfb6bde00b281934e2212bb7529ea3006c0031a556a84d2268bea5", size = 285792 },
    { url = "https://files.pythonhosted.org/packages/b9/8a/fb9d76ecb44a79f02188240278574376e851d0ca81437f433c9e6481d2e5/bitarray-3.0.0-cp311-cp311-manylinux_2_17_ppc64le.manylinux2014_ppc64le.whl", hash = "sha256:2055206ed653bee0b56628f6a4d248d53e5660228d355bbec0014bdfa27050ae", size = 300848 },
    { url = "https://files.pythonhosted.org/packages/63/c5/067b688553b23e99d61ecf930abf1ad5cb5f80c2ebe6f0e2fe8ecab00b3f/bitarray-3.0.0-cp311-cp311-manylinux_2_17_s390x.manylinux2014_s390x.whl", hash = "sha256:147542299f458bdb177f798726e5f7d39ab8491de4182c3c6d9885ed275a3c2b", size = 303027 },
    { url = "https://files.pythonhosted.org/packages/dc/46/25ebc667907736b2c5c84f4bd8260d9bece8b69719a33db5c3f3dcb281a5/bitarray-3.0.0-cp311-cp311-manylinux_2_17_x86_64.manylinux2014_x86_64.whl", hash = "sha256:d3f761184b93092077c7f6b7dad7bd4e671c1620404a76620da7872ceb576a94", size = 286125 },
    { url = "https://files.pythonhosted.org/packages/16/dd/f9a1d84965a992ff42cae5b61536e68fc944f3e31a349b690347d98fc5e0/bitarray-3.0.0-cp311-cp311-manylinux_2_5_i686.manylinux1_i686.manylinux_2_17_i686.manylinux2014_i686.whl", hash = "sha256:e008b7b4ce6c7f7a54b250c45c28d4243cc2a3bbfd5298fa7dac92afda229842", size = 277111 },
    { url = "https://files.pythonhosted.org/packages/16/5b/44f298586a09beb62ec553f9efa06c8a5356d2e230e4080c72cb2800a48f/bitarray-3.0.0-cp311-cp311-musllinux_1_2_aarch64.whl", hash = "sha256:dfea514e665af278b2e1d4deb542de1cd4f77413bee83dd15ae16175976ea8d5", size = 280941 },
    { url = "https://files.pythonhosted.org/packages/28/7c/c6e157332227862727959057ba2987e6710985992b196a81f61995f21e19/bitarray-3.0.0-cp311-cp311-musllinux_1_2_i686.whl", hash = "sha256:66d6134b7bb737b88f1d16478ad0927c571387f6054f4afa5557825a4c1b78e2", size = 272817 },
    { url = "https://files.pythonhosted.org/packages/b7/5d/9f7aaaaf85b5247b4a69b93af60ac7dcfff5545bf544a35517618c4244a0/bitarray-3.0.0-cp311-cp311-musllinux_1_2_ppc64le.whl", hash = "sha256:3cd565253889940b4ec4768d24f101d9fe111cad4606fdb203ea16f9797cf9ed", size = 295830 },
    { url = "https://files.pythonhosted.org/packages/14/1b/86dd50edd2e0612b092fe4caec3001a24298c9acab5e89a503f002ed3bef/bitarray-3.0.0-cp311-cp311-musllinux_1_2_s390x.whl", hash = "sha256:4800c91a14656789d2e67d9513359e23e8a534c8ee1482bb9b517a4cfc845200", size = 307592 },
    { url = "https://files.pythonhosted.org/packages/d6/8f/45a1f1bcce5fd88d2f0bb2e1ebe8bbb55247edcb8e7a8ef06e4437e2b5e3/bitarray-3.0.0-cp311-cp311-musllinux_1_2_x86_64.whl", hash = "sha256:c2945e0390d1329c585c584c6b6d78be017d9c6a1288f9c92006fe907f69cc28", size = 278971 },
    { url = "https://files.pythonhosted.org/packages/43/2d/948c5718fe901aa58c98cef52b8898a6bea865bea7528cff6c2bc703f9f3/bitarray-3.0.0-cp311-cp311-win32.whl", hash = "sha256:c23286abba0cb509733c6ce8f4013cd951672c332b2e184dbefbd7331cd234c8", size = 114242 },
    { url = "https://files.pythonhosted.org/packages/8c/75/e921ada57bb0bcece5eb515927c031f0bc828f702b8f213639358d9df396/bitarray-3.0.0-cp311-cp311-win_amd64.whl", hash = "sha256:ca79f02a98cbda1472449d440592a2fe2ad96fe55515a0447fa8864a38017cf8", size = 121524 },
    { url = "https://files.pythonhosted.org/packages/4e/2e/2e4beb2b714dc83a9e90ac0e4bacb1a191c71125734f72962ee2a20b9cfb/bitarray-3.0.0-cp312-cp312-macosx_10_13_universal2.whl", hash = "sha256:184972c96e1c7e691be60c3792ca1a51dd22b7f25d96ebea502fe3c9b554f25d", size = 172152 },
    { url = "https://files.pythonhosted.org/packages/e0/1f/9ec96408c060ffc3df5ba64d2b520fd0484cb3393a96691df8f660a43b17/bitarray-3.0.0-cp312-cp312-macosx_10_13_x86_64.whl", hash = "sha256:787db8da5e9e29be712f7a6bce153c7bc8697ccc2c38633e347bb9c82475d5c9", size = 123319 },
    { url = "https://files.pythonhosted.org/packages/80/9f/4dd05086308bfcc84ad88c663460a8ad9f5f638f9f96eb5fa08381054db6/bitarray-3.0.0-cp312-cp312-macosx_11_0_arm64.whl", hash = "sha256:2da91ab3633c66999c2a352f0ca9ae064f553e5fc0eca231d28e7e305b83e942", size = 121242 },
    { url = "https://files.pythonhosted.org/packages/55/bb/8865b7380e9d20445bc775079f24f2279a8c0d9ee11d57c49b118d39beaf/bitarray-3.0.0-cp312-cp312-manylinux_2_17_aarch64.manylinux2014_aarch64.whl", hash = "sha256:7edb83089acbf2c86c8002b96599071931dc4ea5e1513e08306f6f7df879a48b", size = 287463 },
    { url = "https://files.pythonhosted.org/packages/db/8b/779119ee438090a80cbfaa49f96e783651183ab4c25b9760fe360aa7cb31/bitarray-3.0.0-cp312-cp312-manylinux_2_17_ppc64le.manylinux2014_ppc64le.whl", hash = "sha256:996d1b83eb904589f40974538223eaed1ab0f62be8a5105c280b9bd849e685c4", size = 301599 },
    { url = "https://files.pythonhosted.org/packages/41/25/78f7ba7fa8ab428767dfb722fc1ea9aac4a9813e348023d8047d8fd32253/bitarray-3.0.0-cp312-cp312-manylinux_2_17_s390x.manylinux2014_s390x.whl", hash = "sha256:4817d73d995bd2b977d9cde6050be8d407791cf1f84c8047fa0bea88c1b815bc", size = 304837 },
    { url = "https://files.pythonhosted.org/packages/f7/8d/30a448d3157b4239e635c92fc3b3789a5b87784875ca2776f65bd543d136/bitarray-3.0.0-cp312-cp312-manylinux_2_17_x86_64.manylinux2014_x86_64.whl", hash = "sha256:3d47bc4ff9b0e1624d613563c6fa7b80aebe7863c56c3df5ab238bb7134e8755", size = 288588 },
    { url = "https://files.pythonhosted.org/packages/86/e0/c1f1b595682244f55119d55f280b5a996bcd462b702ec220d976a7566d27/bitarray-3.0.0-cp312-cp312-manylinux_2_5_i686.manylinux1_i686.manylinux_2_17_i686.manylinux2014_i686.whl", hash = "sha256:aca0a9cd376beaccd9f504961de83e776dd209c2de5a4c78dc87a78edf61839b", size = 279002 },
    { url = "https://files.pythonhosted.org/packages/5c/4d/a17626923ad2c9d20ed1625fc5b27a8dfe2d1a3e877083e9422455ec302d/bitarray-3.0.0-cp312-cp312-musllinux_1_2_aarch64.whl", hash = "sha256:572a61fba7e3a710a8324771322fba8488d134034d349dcd036a7aef74723a80", size = 281898 },
    { url = "https://files.pythonhosted.org/packages/50/d8/5c410580a510e669d9a28bf17675e58843236c55c60fc6dc8f8747808757/bitarray-3.0.0-cp312-cp312-musllinux_1_2_i686.whl", hash = "sha256:a817ad70c1aff217530576b4f037dd9b539eb2926603354fcac605d824082ad1", size = 274622 },
    { url = "https://files.pythonhosted.org/packages/e7/21/de2e8eda85c5f6a05bda75a00c22c94aee71ef09db0d5cbf22446de74312/bitarray-3.0.0-cp312-cp312-musllinux_1_2_ppc64le.whl", hash = "sha256:2ac67b658fa5426503e9581a3fb44a26a3b346c1abd17105735f07db572195b3", size = 296930 },
    { url = "https://files.pythonhosted.org/packages/13/7b/7cfad12d77db2932fb745fa281693b0031c3dfd7f2ecf5803be688cc3798/bitarray-3.0.0-cp312-cp312-musllinux_1_2_s390x.whl", hash = "sha256:12f19ede03e685c5c588ab5ed63167999295ffab5e1126c5fe97d12c0718c18f", size = 309836 },
    { url = "https://files.pythonhosted.org/packages/53/e1/5120fbb8438a0d718e063f70168a2975e03f00ce6b86e74b8eec079cb492/bitarray-3.0.0-cp312-cp312-musllinux_1_2_x86_64.whl", hash = "sha256:fcef31b062f756ba7eebcd7890c5d5de84b9d64ee877325257bcc9782288564a", size = 281535 },
    { url = "https://files.pythonhosted.org/packages/73/75/8acebbbb4f85dcca73b8e91dde5d3e1e3e2317b36fae4f5b133c60720834/bitarray-3.0.0-cp312-cp312-win32.whl", hash = "sha256:656db7bdf1d81ec3b57b3cad7ec7276765964bcfd0eb81c5d1331f385298169c", size = 114423 },
    { url = "https://files.pythonhosted.org/packages/ca/56/dadae4d4351b337de6e0269001fb40f3ebe9f72222190456713d2c1be53d/bitarray-3.0.0-cp312-cp312-win_amd64.whl", hash = "sha256:f785af6b7cb07a9b1e5db0dea9ef9e3e8bb3d74874a0a61303eab9c16acc1999", size = 121680 },
    { url = "https://files.pythonhosted.org/packages/4f/30/07d7be4624981537d32b261dc48a16b03757cc9d88f66012d93acaf11663/bitarray-3.0.0-cp313-cp313-macosx_10_13_universal2.whl", hash = "sha256:7cb885c043000924554fe2124d13084c8fdae03aec52c4086915cd4cb87fe8be", size = 172147 },
    { url = "https://files.pythonhosted.org/packages/f0/e9/be1fa2828bad9cb32e1309e6dbd05adcc41679297d9e96bbb372be928e38/bitarray-3.0.0-cp313-cp313-macosx_10_13_x86_64.whl", hash = "sha256:7814c9924a0b30ecd401f02f082d8697fc5a5be3f8d407efa6e34531ff3c306a", size = 123319 },
    { url = "https://files.pythonhosted.org/packages/22/28/33601d276a6eb76e40fe8a61c61f59cc9ff6d9ecf0b676235c02689475b8/bitarray-3.0.0-cp313-cp313-macosx_11_0_arm64.whl", hash = "sha256:bcf524a087b143ba736aebbb054bb399d49e77cf7c04ed24c728e411adc82bfa", size = 121236 },
    { url = "https://files.pythonhosted.org/packages/85/d3/f36b213ffae8f9c8e4c6f12a91e18c06570a04f42d5a1bda4303380f2639/bitarray-3.0.0-cp313-cp313-manylinux_2_17_aarch64.manylinux2014_aarch64.whl", hash = "sha256:d1d5abf1d6d910599ac16afdd9a0ed3e24f3b46af57f3070cf2792f236f36e0b", size = 287395 },
    { url = "https://files.pythonhosted.org/packages/b7/1a/2da3b00d876883b05ffd3be9b1311858b48d4a26579f8647860e271c5385/bitarray-3.0.0-cp313-cp313-manylinux_2_17_ppc64le.manylinux2014_ppc64le.whl", hash = "sha256:9929051feeaf8d948cc0b1c9ce57748079a941a1a15c89f6014edf18adaade84", size = 301501 },
    { url = "https://files.pythonhosted.org/packages/88/b9/c1b5af8d1c918f1ee98748f7f7270f932f531c2259dd578c0edcf16ec73e/bitarray-3.0.0-cp313-cp313-manylinux_2_17_s390x.manylinux2014_s390x.whl", hash = "sha256:96cf0898f8060b2d3ae491762ae871b071212ded97ff9e1e3a5229e9fefe544c", size = 304804 },
    { url = "https://files.pythonhosted.org/packages/92/24/81a10862856419638c0db13e04de7cbf19938353517a67e4848c691f0b7c/bitarray-3.0.0-cp313-cp313-manylinux_2_17_x86_64.manylinux2014_x86_64.whl", hash = "sha256:ab37da66a8736ad5a75a58034180e92c41e864da0152b84e71fcc253a2f69cd4", size = 288507 },
    { url = "https://files.pythonhosted.org/packages/da/70/a093af92ef7b207a59087e3b5819e03767fbdda9dd56aada3a4ee25a1fbd/bitarray-3.0.0-cp313-cp313-manylinux_2_5_i686.manylinux1_i686.manylinux_2_17_i686.manylinux2014_i686.whl", hash = "sha256:beeb79e476d19b91fd6a3439853e4e5ba1b3b475920fa40d62bde719c8af786f", size = 278905 },
    { url = "https://files.pythonhosted.org/packages/fb/40/0925c6079c4b282b16eb9085f82df0cdf1f787fb4c67fd4baca3e37acf7f/bitarray-3.0.0-cp313-cp313-musllinux_1_2_aarch64.whl", hash = "sha256:f75fc0198c955d840b836059bd43e0993edbf119923029ca60c4fc017cefa54a", size = 281909 },
    { url = "https://files.pythonhosted.org/packages/61/4b/e11754a5d34cb997250d8019b1fe555d4c06fe2d2a68b0bf7c5580537046/bitarray-3.0.0-cp313-cp313-musllinux_1_2_i686.whl", hash = "sha256:f12cc7c7638074918cdcc7491aff897df921b092ffd877227892d2686e98f876", size = 274711 },
    { url = "https://files.pythonhosted.org/packages/5b/78/39513f75423959ee2d82a82e10296b6a7bc7d880b16d714980a6752ef33b/bitarray-3.0.0-cp313-cp313-musllinux_1_2_ppc64le.whl", hash = "sha256:dbe1084935b942fab206e609fa1ed3f46ad1f2612fb4833e177e9b2a5e006c96", size = 297038 },
    { url = "https://files.pythonhosted.org/packages/af/a2/5cb81f8773a479de7c06cc1ada36d5cc5a8ebcd8715013e1c4e01a76e84a/bitarray-3.0.0-cp313-cp313-musllinux_1_2_s390x.whl", hash = "sha256:ac06dd72ee1e1b6e312504d06f75220b5894af1fb58f0c20643698f5122aea76", size = 309814 },
    { url = "https://files.pythonhosted.org/packages/03/3e/795b57c6f6eea61c47d0716e1d60219218028b1f260f7328802eac684964/bitarray-3.0.0-cp313-cp313-musllinux_1_2_x86_64.whl", hash = "sha256:00f9a88c56e373009ac3c73c55205cfbd9683fbd247e2f9a64bae3da78795252", size = 281564 },
    { url = "https://files.pythonhosted.org/packages/f6/31/5914002ae4dd0e0079f8bccfd0647119cff364280d106108a19bd2511933/bitarray-3.0.0-cp313-cp313-win32.whl", hash = "sha256:9c6e52005e91803eb4e08c0a08a481fb55ddce97f926bae1f6fa61b3396b5b61", size = 114404 },
    { url = "https://files.pythonhosted.org/packages/76/0a/184f85a1739db841ae8fbb1d9ec028240d5a351e36abec9cd020de889dab/bitarray-3.0.0-cp313-cp313-win_amd64.whl", hash = "sha256:cb98d5b6eac4b2cf2a5a69f60a9c499844b8bea207059e9fc45c752436e6bb49", size = 121672 },
]

[[package]]
name = "certifi"
version = "2024.8.30"
source = { registry = "https://pypi.org/simple" }
sdist = { url = "https://files.pythonhosted.org/packages/b0/ee/9b19140fe824b367c04c5e1b369942dd754c4c5462d5674002f75c4dedc1/certifi-2024.8.30.tar.gz", hash = "sha256:bec941d2aa8195e248a60b31ff9f0558284cf01a52591ceda73ea9afffd69fd9", size = 168507 }
wheels = [
    { url = "https://files.pythonhosted.org/packages/12/90/3c9ff0512038035f59d279fddeb79f5f1eccd8859f06d6163c58798b9487/certifi-2024.8.30-py3-none-any.whl", hash = "sha256:922820b53db7a7257ffbda3f597266d435245903d80737e34f8a45ff3e3230d8", size = 167321 },
]

[[package]]
name = "charset-normalizer"
version = "3.4.0"
source = { registry = "https://pypi.org/simple" }
sdist = { url = "https://files.pythonhosted.org/packages/f2/4f/e1808dc01273379acc506d18f1504eb2d299bd4131743b9fc54d7be4df1e/charset_normalizer-3.4.0.tar.gz", hash = "sha256:223217c3d4f82c3ac5e29032b3f1c2eb0fb591b72161f86d93f5719079dae93e", size = 106620 }
wheels = [
    { url = "https://files.pythonhosted.org/packages/9c/61/73589dcc7a719582bf56aae309b6103d2762b526bffe189d635a7fcfd998/charset_normalizer-3.4.0-cp311-cp311-macosx_10_9_universal2.whl", hash = "sha256:0d99dd8ff461990f12d6e42c7347fd9ab2532fb70e9621ba520f9e8637161d7c", size = 193339 },
    { url = "https://files.pythonhosted.org/packages/77/d5/8c982d58144de49f59571f940e329ad6e8615e1e82ef84584c5eeb5e1d72/charset_normalizer-3.4.0-cp311-cp311-macosx_10_9_x86_64.whl", hash = "sha256:c57516e58fd17d03ebe67e181a4e4e2ccab1168f8c2976c6a334d4f819fe5944", size = 124366 },
    { url = "https://files.pythonhosted.org/packages/bf/19/411a64f01ee971bed3231111b69eb56f9331a769072de479eae7de52296d/charset_normalizer-3.4.0-cp311-cp311-macosx_11_0_arm64.whl", hash = "sha256:6dba5d19c4dfab08e58d5b36304b3f92f3bd5d42c1a3fa37b5ba5cdf6dfcbcee", size = 118874 },
    { url = "https://files.pythonhosted.org/packages/4c/92/97509850f0d00e9f14a46bc751daabd0ad7765cff29cdfb66c68b6dad57f/charset_normalizer-3.4.0-cp311-cp311-manylinux_2_17_aarch64.manylinux2014_aarch64.whl", hash = "sha256:bf4475b82be41b07cc5e5ff94810e6a01f276e37c2d55571e3fe175e467a1a1c", size = 138243 },
    { url = "https://files.pythonhosted.org/packages/e2/29/d227805bff72ed6d6cb1ce08eec707f7cfbd9868044893617eb331f16295/charset_normalizer-3.4.0-cp311-cp311-manylinux_2_17_ppc64le.manylinux2014_ppc64le.whl", hash = "sha256:ce031db0408e487fd2775d745ce30a7cd2923667cf3b69d48d219f1d8f5ddeb6", size = 148676 },
    { url = "https://files.pythonhosted.org/packages/13/bc/87c2c9f2c144bedfa62f894c3007cd4530ba4b5351acb10dc786428a50f0/charset_normalizer-3.4.0-cp311-cp311-manylinux_2_17_s390x.manylinux2014_s390x.whl", hash = "sha256:8ff4e7cdfdb1ab5698e675ca622e72d58a6fa2a8aa58195de0c0061288e6e3ea", size = 141289 },
    { url = "https://files.pythonhosted.org/packages/eb/5b/6f10bad0f6461fa272bfbbdf5d0023b5fb9bc6217c92bf068fa5a99820f5/charset_normalizer-3.4.0-cp311-cp311-manylinux_2_17_x86_64.manylinux2014_x86_64.whl", hash = "sha256:3710a9751938947e6327ea9f3ea6332a09bf0ba0c09cae9cb1f250bd1f1549bc", size = 142585 },
    { url = "https://files.pythonhosted.org/packages/3b/a0/a68980ab8a1f45a36d9745d35049c1af57d27255eff8c907e3add84cf68f/charset_normalizer-3.4.0-cp311-cp311-manylinux_2_5_i686.manylinux1_i686.manylinux_2_17_i686.manylinux2014_i686.whl", hash = "sha256:82357d85de703176b5587dbe6ade8ff67f9f69a41c0733cf2425378b49954de5", size = 144408 },
    { url = "https://files.pythonhosted.org/packages/d7/a1/493919799446464ed0299c8eef3c3fad0daf1c3cd48bff9263c731b0d9e2/charset_normalizer-3.4.0-cp311-cp311-musllinux_1_2_aarch64.whl", hash = "sha256:47334db71978b23ebcf3c0f9f5ee98b8d65992b65c9c4f2d34c2eaf5bcaf0594", size = 139076 },
    { url = "https://files.pythonhosted.org/packages/fb/9d/9c13753a5a6e0db4a0a6edb1cef7aee39859177b64e1a1e748a6e3ba62c2/charset_normalizer-3.4.0-cp311-cp311-musllinux_1_2_i686.whl", hash = "sha256:8ce7fd6767a1cc5a92a639b391891bf1c268b03ec7e021c7d6d902285259685c", size = 146874 },
    { url = "https://files.pythonhosted.org/packages/75/d2/0ab54463d3410709c09266dfb416d032a08f97fd7d60e94b8c6ef54ae14b/charset_normalizer-3.4.0-cp311-cp311-musllinux_1_2_ppc64le.whl", hash = "sha256:f1a2f519ae173b5b6a2c9d5fa3116ce16e48b3462c8b96dfdded11055e3d6365", size = 150871 },
    { url = "https://files.pythonhosted.org/packages/8d/c9/27e41d481557be53d51e60750b85aa40eaf52b841946b3cdeff363105737/charset_normalizer-3.4.0-cp311-cp311-musllinux_1_2_s390x.whl", hash = "sha256:63bc5c4ae26e4bc6be6469943b8253c0fd4e4186c43ad46e713ea61a0ba49129", size = 148546 },
    { url = "https://files.pythonhosted.org/packages/ee/44/4f62042ca8cdc0cabf87c0fc00ae27cd8b53ab68be3605ba6d071f742ad3/charset_normalizer-3.4.0-cp311-cp311-musllinux_1_2_x86_64.whl", hash = "sha256:bcb4f8ea87d03bc51ad04add8ceaf9b0f085ac045ab4d74e73bbc2dc033f0236", size = 143048 },
    { url = "https://files.pythonhosted.org/packages/01/f8/38842422988b795220eb8038745d27a675ce066e2ada79516c118f291f07/charset_normalizer-3.4.0-cp311-cp311-win32.whl", hash = "sha256:9ae4ef0b3f6b41bad6366fb0ea4fc1d7ed051528e113a60fa2a65a9abb5b1d99", size = 94389 },
    { url = "https://files.pythonhosted.org/packages/0b/6e/b13bd47fa9023b3699e94abf565b5a2f0b0be6e9ddac9812182596ee62e4/charset_normalizer-3.4.0-cp311-cp311-win_amd64.whl", hash = "sha256:cee4373f4d3ad28f1ab6290684d8e2ebdb9e7a1b74fdc39e4c211995f77bec27", size = 101752 },
    { url = "https://files.pythonhosted.org/packages/d3/0b/4b7a70987abf9b8196845806198975b6aab4ce016632f817ad758a5aa056/charset_normalizer-3.4.0-cp312-cp312-macosx_10_13_universal2.whl", hash = "sha256:0713f3adb9d03d49d365b70b84775d0a0d18e4ab08d12bc46baa6132ba78aaf6", size = 194445 },
    { url = "https://files.pythonhosted.org/packages/50/89/354cc56cf4dd2449715bc9a0f54f3aef3dc700d2d62d1fa5bbea53b13426/charset_normalizer-3.4.0-cp312-cp312-macosx_10_13_x86_64.whl", hash = "sha256:de7376c29d95d6719048c194a9cf1a1b0393fbe8488a22008610b0361d834ecf", size = 125275 },
    { url = "https://files.pythonhosted.org/packages/fa/44/b730e2a2580110ced837ac083d8ad222343c96bb6b66e9e4e706e4d0b6df/charset_normalizer-3.4.0-cp312-cp312-macosx_11_0_arm64.whl", hash = "sha256:4a51b48f42d9358460b78725283f04bddaf44a9358197b889657deba38f329db", size = 119020 },
    { url = "https://files.pythonhosted.org/packages/9d/e4/9263b8240ed9472a2ae7ddc3e516e71ef46617fe40eaa51221ccd4ad9a27/charset_normalizer-3.4.0-cp312-cp312-manylinux_2_17_aarch64.manylinux2014_aarch64.whl", hash = "sha256:b295729485b06c1a0683af02a9e42d2caa9db04a373dc38a6a58cdd1e8abddf1", size = 139128 },
    { url = "https://files.pythonhosted.org/packages/6b/e3/9f73e779315a54334240353eaea75854a9a690f3f580e4bd85d977cb2204/charset_normalizer-3.4.0-cp312-cp312-manylinux_2_17_ppc64le.manylinux2014_ppc64le.whl", hash = "sha256:ee803480535c44e7f5ad00788526da7d85525cfefaf8acf8ab9a310000be4b03", size = 149277 },
    { url = "https://files.pythonhosted.org/packages/1a/cf/f1f50c2f295312edb8a548d3fa56a5c923b146cd3f24114d5adb7e7be558/charset_normalizer-3.4.0-cp312-cp312-manylinux_2_17_s390x.manylinux2014_s390x.whl", hash = "sha256:3d59d125ffbd6d552765510e3f31ed75ebac2c7470c7274195b9161a32350284", size = 142174 },
    { url = "https://files.pythonhosted.org/packages/16/92/92a76dc2ff3a12e69ba94e7e05168d37d0345fa08c87e1fe24d0c2a42223/charset_normalizer-3.4.0-cp312-cp312-manylinux_2_17_x86_64.manylinux2014_x86_64.whl", hash = "sha256:8cda06946eac330cbe6598f77bb54e690b4ca93f593dee1568ad22b04f347c15", size = 143838 },
    { url = "https://files.pythonhosted.org/packages/a4/01/2117ff2b1dfc61695daf2babe4a874bca328489afa85952440b59819e9d7/charset_normalizer-3.4.0-cp312-cp312-manylinux_2_5_i686.manylinux1_i686.manylinux_2_17_i686.manylinux2014_i686.whl", hash = "sha256:07afec21bbbbf8a5cc3651aa96b980afe2526e7f048fdfb7f1014d84acc8b6d8", size = 146149 },
    { url = "https://files.pythonhosted.org/packages/f6/9b/93a332b8d25b347f6839ca0a61b7f0287b0930216994e8bf67a75d050255/charset_normalizer-3.4.0-cp312-cp312-musllinux_1_2_aarch64.whl", hash = "sha256:6b40e8d38afe634559e398cc32b1472f376a4099c75fe6299ae607e404c033b2", size = 140043 },
    { url = "https://files.pythonhosted.org/packages/ab/f6/7ac4a01adcdecbc7a7587767c776d53d369b8b971382b91211489535acf0/charset_normalizer-3.4.0-cp312-cp312-musllinux_1_2_i686.whl", hash = "sha256:b8dcd239c743aa2f9c22ce674a145e0a25cb1566c495928440a181ca1ccf6719", size = 148229 },
    { url = "https://files.pythonhosted.org/packages/9d/be/5708ad18161dee7dc6a0f7e6cf3a88ea6279c3e8484844c0590e50e803ef/charset_normalizer-3.4.0-cp312-cp312-musllinux_1_2_ppc64le.whl", hash = "sha256:84450ba661fb96e9fd67629b93d2941c871ca86fc38d835d19d4225ff946a631", size = 151556 },
    { url = "https://files.pythonhosted.org/packages/5a/bb/3d8bc22bacb9eb89785e83e6723f9888265f3a0de3b9ce724d66bd49884e/charset_normalizer-3.4.0-cp312-cp312-musllinux_1_2_s390x.whl", hash = "sha256:44aeb140295a2f0659e113b31cfe92c9061622cadbc9e2a2f7b8ef6b1e29ef4b", size = 149772 },
    { url = "https://files.pythonhosted.org/packages/f7/fa/d3fc622de05a86f30beea5fc4e9ac46aead4731e73fd9055496732bcc0a4/charset_normalizer-3.4.0-cp312-cp312-musllinux_1_2_x86_64.whl", hash = "sha256:1db4e7fefefd0f548d73e2e2e041f9df5c59e178b4c72fbac4cc6f535cfb1565", size = 144800 },
    { url = "https://files.pythonhosted.org/packages/9a/65/bdb9bc496d7d190d725e96816e20e2ae3a6fa42a5cac99c3c3d6ff884118/charset_normalizer-3.4.0-cp312-cp312-win32.whl", hash = "sha256:5726cf76c982532c1863fb64d8c6dd0e4c90b6ece9feb06c9f202417a31f7dd7", size = 94836 },
    { url = "https://files.pythonhosted.org/packages/3e/67/7b72b69d25b89c0b3cea583ee372c43aa24df15f0e0f8d3982c57804984b/charset_normalizer-3.4.0-cp312-cp312-win_amd64.whl", hash = "sha256:b197e7094f232959f8f20541ead1d9862ac5ebea1d58e9849c1bf979255dfac9", size = 102187 },
    { url = "https://files.pythonhosted.org/packages/f3/89/68a4c86f1a0002810a27f12e9a7b22feb198c59b2f05231349fbce5c06f4/charset_normalizer-3.4.0-cp313-cp313-macosx_10_13_universal2.whl", hash = "sha256:dd4eda173a9fcccb5f2e2bd2a9f423d180194b1bf17cf59e3269899235b2a114", size = 194617 },
    { url = "https://files.pythonhosted.org/packages/4f/cd/8947fe425e2ab0aa57aceb7807af13a0e4162cd21eee42ef5b053447edf5/charset_normalizer-3.4.0-cp313-cp313-macosx_10_13_x86_64.whl", hash = "sha256:e9e3c4c9e1ed40ea53acf11e2a386383c3304212c965773704e4603d589343ed", size = 125310 },
    { url = "https://files.pythonhosted.org/packages/5b/f0/b5263e8668a4ee9becc2b451ed909e9c27058337fda5b8c49588183c267a/charset_normalizer-3.4.0-cp313-cp313-macosx_11_0_arm64.whl", hash = "sha256:92a7e36b000bf022ef3dbb9c46bfe2d52c047d5e3f3343f43204263c5addc250", size = 119126 },
    { url = "https://files.pythonhosted.org/packages/ff/6e/e445afe4f7fda27a533f3234b627b3e515a1b9429bc981c9a5e2aa5d97b6/charset_normalizer-3.4.0-cp313-cp313-manylinux_2_17_aarch64.manylinux2014_aarch64.whl", hash = "sha256:54b6a92d009cbe2fb11054ba694bc9e284dad30a26757b1e372a1fdddaf21920", size = 139342 },
    { url = "https://files.pythonhosted.org/packages/a1/b2/4af9993b532d93270538ad4926c8e37dc29f2111c36f9c629840c57cd9b3/charset_normalizer-3.4.0-cp313-cp313-manylinux_2_17_ppc64le.manylinux2014_ppc64le.whl", hash = "sha256:1ffd9493de4c922f2a38c2bf62b831dcec90ac673ed1ca182fe11b4d8e9f2a64", size = 149383 },
    { url = "https://files.pythonhosted.org/packages/fb/6f/4e78c3b97686b871db9be6f31d64e9264e889f8c9d7ab33c771f847f79b7/charset_normalizer-3.4.0-cp313-cp313-manylinux_2_17_s390x.manylinux2014_s390x.whl", hash = "sha256:35c404d74c2926d0287fbd63ed5d27eb911eb9e4a3bb2c6d294f3cfd4a9e0c23", size = 142214 },
    { url = "https://files.pythonhosted.org/packages/2b/c9/1c8fe3ce05d30c87eff498592c89015b19fade13df42850aafae09e94f35/charset_normalizer-3.4.0-cp313-cp313-manylinux_2_17_x86_64.manylinux2014_x86_64.whl", hash = "sha256:4796efc4faf6b53a18e3d46343535caed491776a22af773f366534056c4e1fbc", size = 144104 },
    { url = "https://files.pythonhosted.org/packages/ee/68/efad5dcb306bf37db7db338338e7bb8ebd8cf38ee5bbd5ceaaaa46f257e6/charset_normalizer-3.4.0-cp313-cp313-manylinux_2_5_i686.manylinux1_i686.manylinux_2_17_i686.manylinux2014_i686.whl", hash = "sha256:e7fdd52961feb4c96507aa649550ec2a0d527c086d284749b2f582f2d40a2e0d", size = 146255 },
    { url = "https://files.pythonhosted.org/packages/0c/75/1ed813c3ffd200b1f3e71121c95da3f79e6d2a96120163443b3ad1057505/charset_normalizer-3.4.0-cp313-cp313-musllinux_1_2_aarch64.whl", hash = "sha256:92db3c28b5b2a273346bebb24857fda45601aef6ae1c011c0a997106581e8a88", size = 140251 },
    { url = "https://files.pythonhosted.org/packages/7d/0d/6f32255c1979653b448d3c709583557a4d24ff97ac4f3a5be156b2e6a210/charset_normalizer-3.4.0-cp313-cp313-musllinux_1_2_i686.whl", hash = "sha256:ab973df98fc99ab39080bfb0eb3a925181454d7c3ac8a1e695fddfae696d9e90", size = 148474 },
    { url = "https://files.pythonhosted.org/packages/ac/a0/c1b5298de4670d997101fef95b97ac440e8c8d8b4efa5a4d1ef44af82f0d/charset_normalizer-3.4.0-cp313-cp313-musllinux_1_2_ppc64le.whl", hash = "sha256:4b67fdab07fdd3c10bb21edab3cbfe8cf5696f453afce75d815d9d7223fbe88b", size = 151849 },
    { url = "https://files.pythonhosted.org/packages/04/4f/b3961ba0c664989ba63e30595a3ed0875d6790ff26671e2aae2fdc28a399/charset_normalizer-3.4.0-cp313-cp313-musllinux_1_2_s390x.whl", hash = "sha256:aa41e526a5d4a9dfcfbab0716c7e8a1b215abd3f3df5a45cf18a12721d31cb5d", size = 149781 },
    { url = "https://files.pythonhosted.org/packages/d8/90/6af4cd042066a4adad58ae25648a12c09c879efa4849c705719ba1b23d8c/charset_normalizer-3.4.0-cp313-cp313-musllinux_1_2_x86_64.whl", hash = "sha256:ffc519621dce0c767e96b9c53f09c5d215578e10b02c285809f76509a3931482", size = 144970 },
    { url = "https://files.pythonhosted.org/packages/cc/67/e5e7e0cbfefc4ca79025238b43cdf8a2037854195b37d6417f3d0895c4c2/charset_normalizer-3.4.0-cp313-cp313-win32.whl", hash = "sha256:f19c1585933c82098c2a520f8ec1227f20e339e33aca8fa6f956f6691b784e67", size = 94973 },
    { url = "https://files.pythonhosted.org/packages/65/97/fc9bbc54ee13d33dc54a7fcf17b26368b18505500fc01e228c27b5222d80/charset_normalizer-3.4.0-cp313-cp313-win_amd64.whl", hash = "sha256:707b82d19e65c9bd28b81dde95249b07bf9f5b90ebe1ef17d9b57473f8a64b7b", size = 102308 },
    { url = "https://files.pythonhosted.org/packages/bf/9b/08c0432272d77b04803958a4598a51e2a4b51c06640af8b8f0f908c18bf2/charset_normalizer-3.4.0-py3-none-any.whl", hash = "sha256:fe9f97feb71aa9896b81973a7bbada8c49501dc73e58a10fcef6663af95e5079", size = 49446 },
]

[[package]]
name = "ckzg"
version = "2.0.1"
source = { registry = "https://pypi.org/simple" }
sdist = { url = "https://files.pythonhosted.org/packages/70/80/4b6219a65634915efc4694fa606f38d4b893dcdc1e50b9bcf69b38ec82b0/ckzg-2.0.1.tar.gz", hash = "sha256:62c5adc381637affa7e1df465c57750b356a761b8a3164c3106589b02532b9c9", size = 1113747 }
wheels = [
    { url = "https://files.pythonhosted.org/packages/56/eb/00e5978a32facf9203e2069d7905f8a5c262cafd6795adf9933531e7427f/ckzg-2.0.1-cp311-cp311-macosx_10_9_x86_64.whl", hash = "sha256:260608a22e2f2cadcd31f4495832d45d6460438c38faba9761b92df885a99d88", size = 114743 },
    { url = "https://files.pythonhosted.org/packages/b7/99/65015b5c6447293a3321112cb33d303ae07a5000302fa08c976819fcad64/ckzg-2.0.1-cp311-cp311-macosx_11_0_arm64.whl", hash = "sha256:e1015f99c50215098751b07d7e459ba9a2790d3692ca81552eed29996128e90d", size = 98687 },
    { url = "https://files.pythonhosted.org/packages/92/ec/5d324b490b30e581888d8f7243c3259aad6a25913653ea6d8d673c84ec0d/ckzg-2.0.1-cp311-cp311-manylinux_2_12_i686.manylinux2010_i686.manylinux_2_17_i686.manylinux2014_i686.whl", hash = "sha256:9dd350d97554c161dc5b8c7b32c2dc8e659632c374f60e2669fb3c9b5b294827", size = 174848 },
    { url = "https://files.pythonhosted.org/packages/71/50/40a6c567c15eb65c73cbc45c1ed9341a0f1ea77bf489078f56e40e0e4836/ckzg-2.0.1-cp311-cp311-manylinux_2_17_aarch64.manylinux2014_aarch64.whl", hash = "sha256:eec7724fa8dc4ae95757efe4a87e7b2d4b880cb348c72ce7355fc0c4f64bc298", size = 160846 },
    { url = "https://files.pythonhosted.org/packages/03/c9/044fbe6083fc0e8477b9545b193b82bf349a03f4b561f87870f0b2280f36/ckzg-2.0.1-cp311-cp311-manylinux_2_17_x86_64.manylinux2014_x86_64.whl", hash = "sha256:c3fa0f4398fa67fb71f0a2b34a652cc89e6e0e6af1340b0dc771db1a5f3e089c", size = 169632 },
    { url = "https://files.pythonhosted.org/packages/ae/f5/54e736a969813c72c745c2f0c2906e973d63f89744d947c4e3429c81761d/ckzg-2.0.1-cp311-cp311-musllinux_1_2_aarch64.whl", hash = "sha256:f865a0297aabeeb638187a46f7df445763360417b9df4dea60560d512c2cda09", size = 171214 },
    { url = "https://files.pythonhosted.org/packages/f4/2f/0896c133021479c5fb57ffbddd584774bdebd7d8c966be732380aa1d62ac/ckzg-2.0.1-cp311-cp311-musllinux_1_2_i686.whl", hash = "sha256:5b6ec738350771dbf5974fb70cc8bbb20a4df784af770f7e655922adc08a2171", size = 185708 },
    { url = "https://files.pythonhosted.org/packages/02/a8/513876410e9634f10ef933f6aae6e71afd3e0786817773c9d40908b6abd4/ckzg-2.0.1-cp311-cp311-musllinux_1_2_x86_64.whl", hash = "sha256:9b4b669fc77edeb16adc182efc32b3737b36f741a2e33a170d40619e8b171a94", size = 179787 },
    { url = "https://files.pythonhosted.org/packages/c4/b6/5fe04bdbcf881bf23626ff82145e0c077ee3bdb35c86d177bf37fa57832e/ckzg-2.0.1-cp311-cp311-win_amd64.whl", hash = "sha256:decb97f4a17c7338b2130dcc4b045df4cc0e7785ece872c764b554c7c73a99ff", size = 98228 },
    { url = "https://files.pythonhosted.org/packages/6c/87/dcc62fc2f6651127b6306a37db492998c291ad1a09a6a0d18895882fec51/ckzg-2.0.1-cp312-cp312-macosx_10_9_x86_64.whl", hash = "sha256:285cf3121b8a8c5609c5b706314f68d2ba2784ab02c5bb7487c6ae1714ecb27f", size = 114776 },
    { url = "https://files.pythonhosted.org/packages/fd/99/2d3aa09ebf692c26e03d17b9e7426a34fd71fe4d9b2ff1acf482736cc8da/ckzg-2.0.1-cp312-cp312-macosx_11_0_arm64.whl", hash = "sha256:2f927bc41c2551b0ef0056a649a7ebed29d9665680a10795f4cee5002c69ddb7", size = 98711 },
    { url = "https://files.pythonhosted.org/packages/50/b3/44a533895aa4257d0dcb2818f7dd9b1321664784cac2d381022ed8c40113/ckzg-2.0.1-cp312-cp312-manylinux_2_12_i686.manylinux2010_i686.manylinux_2_17_i686.manylinux2014_i686.whl", hash = "sha256:1fd9fb690c88919f30c9f3ab7cc46a7ecd734d5ff4c9ccea383c119b9b7cc4da", size = 175026 },
    { url = "https://files.pythonhosted.org/packages/54/a2/c594861665851f91ae81ec29cf90e38999de042aa95604737d4b779a8609/ckzg-2.0.1-cp312-cp312-manylinux_2_17_aarch64.manylinux2014_aarch64.whl", hash = "sha256:fabc3bd41b306d1c7025d561c3281a007c2aca8ceaf998582dc3894904d9c73e", size = 161039 },
    { url = "https://files.pythonhosted.org/packages/59/a0/96bb77fb8bf4cd4d51d8bd1d67d59d13f51fa2477b11b915ab6465aa92ce/ckzg-2.0.1-cp312-cp312-manylinux_2_17_x86_64.manylinux2014_x86_64.whl", hash = "sha256:2eb50c53efdb9c34f762bd0c8006cf79bc92a9daf47aa6b541e496988484124f", size = 169889 },
    { url = "https://files.pythonhosted.org/packages/68/c4/77d54a7e5f85d833e9664935f6278fbea7de30f4fde213d121f7fdbc27a0/ckzg-2.0.1-cp312-cp312-musllinux_1_2_aarch64.whl", hash = "sha256:7960cc62f959403293fb53a3c2404778369ae7cefc6d7f202e5e00567cf98c4b", size = 171378 },
    { url = "https://files.pythonhosted.org/packages/02/54/6520ab37c06680910f8ff99afdc473c945c37ab1016662288d98a028d775/ckzg-2.0.1-cp312-cp312-musllinux_1_2_i686.whl", hash = "sha256:d721bcd492294c70eca39da0b0a433c29b6a571dbac2f7084bab06334904af06", size = 185969 },
    { url = "https://files.pythonhosted.org/packages/d6/fa/16c3a4fd8353a3a9f95728f4141b2800b08e588522f7b5644c91308f6fe1/ckzg-2.0.1-cp312-cp312-musllinux_1_2_x86_64.whl", hash = "sha256:dde2391d025b5033ef0eeacf62b11ecfe446aea25682b5f547a907766ad0a8cb", size = 180093 },
    { url = "https://files.pythonhosted.org/packages/d5/ae/91d36445c247a8832bbb7a71bd75293c4c006731d03a2ccaa13e5506ac8a/ckzg-2.0.1-cp312-cp312-win_amd64.whl", hash = "sha256:fab8859d9420f6f7df4e094ee3639bc49d18c8dab0df81bee825e2363dd67a09", size = 98280 },
    { url = "https://files.pythonhosted.org/packages/79/92/4910b9131eb637c6f72e655c1535b9a9c72a5fb2bdf52742f50066cb9e6b/ckzg-2.0.1-cp313-cp313-macosx_10_13_x86_64.whl", hash = "sha256:9747d92883199d4f8f3a3d7018134745fddcf692dfe67115434e4b32609ea785", size = 114793 },
    { url = "https://files.pythonhosted.org/packages/09/95/cb52623cbc4573b2e65bd924524f479e6a8611002c4634dfd6e9d490b403/ckzg-2.0.1-cp313-cp313-macosx_11_0_arm64.whl", hash = "sha256:b2cf58fb9e165da97f0ffe9f4a6efb73992645fac8e0fa223a6cc7ec486a434a", size = 98715 },
    { url = "https://files.pythonhosted.org/packages/27/05/3f246149a728b5d829dd8e0b75379fd6bce0d420de4042b8ca692083f96d/ckzg-2.0.1-cp313-cp313-manylinux_2_12_i686.manylinux2010_i686.manylinux_2_17_i686.manylinux2014_i686.whl", hash = "sha256:d25d006899d76bb8c9d3e8b27981dd6b66a78f9826e33c1bf981af6577a69a19", size = 175008 },
    { url = "https://files.pythonhosted.org/packages/5f/76/df568b24de6bdbb99fe2c48519744d01f1b9e152fa791ed84b43a2752e78/ckzg-2.0.1-cp313-cp313-manylinux_2_17_aarch64.manylinux2014_aarch64.whl", hash = "sha256:a04bf0b32f04f5ea5e4b8518e292d3321bc05596fde95f9c3b4f504e5e4bc780", size = 161009 },
    { url = "https://files.pythonhosted.org/packages/7f/b5/8bbde8acb339a018c0456fb9af714fcca86ed9bf96114ece9556415afbac/ckzg-2.0.1-cp313-cp313-manylinux_2_17_x86_64.manylinux2014_x86_64.whl", hash = "sha256:1d0cf3dccd72376bff10e1833641cc9d642f34f60ca63972626d9dfcfdc8e77f", size = 169873 },
    { url = "https://files.pythonhosted.org/packages/5f/8f/9b9492f807acbfe791c4c447bbeb96e19160fda272328a9dc6700a2fcb08/ckzg-2.0.1-cp313-cp313-musllinux_1_2_aarch64.whl", hash = "sha256:770809c7e93087470cc524724419b0f85590edb033c7c73ba94aef70b36ca18b", size = 171438 },
    { url = "https://files.pythonhosted.org/packages/f1/58/47d4ed23e338dbe1b06cca99e55ae49c3a539d88576c5893e8b589bf3ac6/ckzg-2.0.1-cp313-cp313-musllinux_1_2_i686.whl", hash = "sha256:e31b59b8124148d5e21f7e41b35532d7af98260c44a77c3917958adece84296d", size = 186020 },
    { url = "https://files.pythonhosted.org/packages/a5/f0/dc6f961b325d186af1a469e7119a0f48cdc36240f2aca6e10a5e5f91b8b8/ckzg-2.0.1-cp313-cp313-musllinux_1_2_x86_64.whl", hash = "sha256:174f0c356df644d6e349ce03b7284d83dbec859e11ca5d1b1b3bace8b8fbc65d", size = 180132 },
    { url = "https://files.pythonhosted.org/packages/72/ca/44676731ca52e6d2289f7e9c74d836f59dc986e9b4182ddd2c7d0b14d88f/ckzg-2.0.1-cp313-cp313-win_amd64.whl", hash = "sha256:30e375cd45142e56b5dbfdec05ce4deb2368d7f7dedfc7408ba37d5639af05ff", size = 98284 },
]

[[package]]
name = "click"
version = "8.1.7"
source = { registry = "https://pypi.org/simple" }
dependencies = [
    { name = "colorama", marker = "platform_system == 'Windows'" },
]
sdist = { url = "https://files.pythonhosted.org/packages/96/d3/f04c7bfcf5c1862a2a5b845c6b2b360488cf47af55dfa79c98f6a6bf98b5/click-8.1.7.tar.gz", hash = "sha256:ca9853ad459e787e2192211578cc907e7594e294c7ccc834310722b41b9ca6de", size = 336121 }
wheels = [
    { url = "https://files.pythonhosted.org/packages/00/2e/d53fa4befbf2cfa713304affc7ca780ce4fc1fd8710527771b58311a3229/click-8.1.7-py3-none-any.whl", hash = "sha256:ae74fb96c20a0277a1d615f1e4d73c8414f5a98db8b799a7931d1582f3390c28", size = 97941 },
]

[[package]]
name = "colorama"
version = "0.4.6"
source = { registry = "https://pypi.org/simple" }
sdist = { url = "https://files.pythonhosted.org/packages/d8/53/6f443c9a4a8358a93a6792e2acffb9d9d5cb0a5cfd8802644b7b1c9a02e4/colorama-0.4.6.tar.gz", hash = "sha256:08695f5cb7ed6e0531a20572697297273c47b8cae5a63ffc6d6ed5c201be6e44", size = 27697 }
wheels = [
    { url = "https://files.pythonhosted.org/packages/d1/d6/3965ed04c63042e047cb6a3e6ed1a63a35087b6a609aa3a15ed8ac56c221/colorama-0.4.6-py2.py3-none-any.whl", hash = "sha256:4f1d9991f5acc0ca119f9d443620b77f9d6b33703e51011c16baf57afb285fc6", size = 25335 },
]

[[package]]
name = "cytoolz"
version = "1.0.0"
source = { registry = "https://pypi.org/simple" }
dependencies = [
    { name = "toolz" },
]
sdist = { url = "https://files.pythonhosted.org/packages/e2/4c/ca9b05bdfa28ddbb4a5365c27021a1d4be61db7d8f6b4e5d4e76aa4ba3b7/cytoolz-1.0.0.tar.gz", hash = "sha256:eb453b30182152f9917a5189b7d99046b6ce90cdf8aeb0feff4b2683e600defd", size = 626708 }
wheels = [
    { url = "https://files.pythonhosted.org/packages/bc/99/b489081777ad02c9bba294c757583416d0bdbd9403017145aba68145c16f/cytoolz-1.0.0-cp311-cp311-macosx_10_9_x86_64.whl", hash = "sha256:dffc22fd2c91be64dbdbc462d0786f8e8ac9a275cfa1869a1084d1867d4f67e0", size = 406148 },
    { url = "https://files.pythonhosted.org/packages/b2/d3/a4d58bf89924dbca34e8dbb643b26935e08c16b4a2ee255d43a8b7489939/cytoolz-1.0.0-cp311-cp311-macosx_11_0_arm64.whl", hash = "sha256:a99e7e29274e293f4ffe20e07f76c2ac753a78f1b40c1828dfc54b2981b2f6c4", size = 384956 },
    { url = "https://files.pythonhosted.org/packages/81/d4/4d09e6571ef3b143f668c590a7a00c97ff24e6df6901f457ea7c782cd2ba/cytoolz-1.0.0-cp311-cp311-manylinux_2_17_aarch64.manylinux2014_aarch64.whl", hash = "sha256:c507a3e0a45c41d66b43f96797290d75d1e7a8549aa03a4a6b8854fdf3f7b8d8", size = 2091688 },
    { url = "https://files.pythonhosted.org/packages/8d/7b/68c89bed2e0490e9b946574c3bc79711179f35b1dc5eb31046c535f1bed2/cytoolz-1.0.0-cp311-cp311-manylinux_2_17_ppc64le.manylinux2014_ppc64le.whl", hash = "sha256:643a593ec272ef7429099e1182a22f64ec2696c00d295d2a5be390db1b7ff176", size = 2188448 },
    { url = "https://files.pythonhosted.org/packages/56/a3/4e536fc7b72fd7495e19180463e8160a4fe1d50ab59a5854fc596621d5c3/cytoolz-1.0.0-cp311-cp311-manylinux_2_17_s390x.manylinux2014_s390x.whl", hash = "sha256:6ce38e2e42cbae30446190c59b92a8a9029e1806fd79eaf88f48b0fe33003893", size = 2174196 },
    { url = "https://files.pythonhosted.org/packages/5a/7f/0451778af9e22755a95ef4400ee7fc6e41387521ab0f17699593cb07169a/cytoolz-1.0.0-cp311-cp311-manylinux_2_17_x86_64.manylinux2014_x86_64.whl", hash = "sha256:810a6a168b8c5ecb412fbae3dd6f7ed6c6253a63caf4174ee9794ebd29b2224f", size = 2099823 },
    { url = "https://files.pythonhosted.org/packages/58/b7/8ffdef1ac8f74b0cc650b9d4a74d93d911a7e20fcf7cc0abac0f4bce225f/cytoolz-1.0.0-cp311-cp311-manylinux_2_5_i686.manylinux1_i686.manylinux_2_17_i686.manylinux2014_i686.whl", hash = "sha256:0ce8a2a85c0741c1b19b16e6782c4a5abc54c3caecda66793447112ab2fa9884", size = 1996733 },
    { url = "https://files.pythonhosted.org/packages/1c/ab/9c694c883f3038d167b797cc55c64c2bdb64146428000cea15f235f30a0f/cytoolz-1.0.0-cp311-cp311-musllinux_1_2_aarch64.whl", hash = "sha256:ea4ac72e6b830861035c4c7999af8e55813f57c6d1913a3d93cc4a6babc27bf7", size = 2013725 },
    { url = "https://files.pythonhosted.org/packages/6c/df/859faaee91c795dc969c79cd38159031f373828d44b0b18999feb7d9a44d/cytoolz-1.0.0-cp311-cp311-musllinux_1_2_i686.whl", hash = "sha256:a09cdfb21dfb38aa04df43e7546a41f673377eb5485da88ceb784e327ec7603b", size = 1994851 },
    { url = "https://files.pythonhosted.org/packages/34/2a/26ac5a34e859c5ba32351f5a74492f4ed12e7a7e75b6afccf11c4100aa70/cytoolz-1.0.0-cp311-cp311-musllinux_1_2_ppc64le.whl", hash = "sha256:658dd85deb375ff7af990a674e5c9058cef1c9d1f5dc89bc87b77be499348144", size = 2155343 },
    { url = "https://files.pythonhosted.org/packages/3d/41/f687d2e40407b29bfcce36a7d456dad368283ea543aa39da53bcc119974e/cytoolz-1.0.0-cp311-cp311-musllinux_1_2_s390x.whl", hash = "sha256:9715d1ff5576919d10b68f17241375f6a1eec8961c25b78a83e6ef1487053f39", size = 2163507 },
    { url = "https://files.pythonhosted.org/packages/39/7c/70d529f909d97ea214d59923c19e3d05a3768fe8e2066542b72550a31ca4/cytoolz-1.0.0-cp311-cp311-musllinux_1_2_x86_64.whl", hash = "sha256:f370a1f1f1afc5c1c8cc5edc1cfe0ba444263a0772af7ce094be8e734f41769d", size = 2054428 },
    { url = "https://files.pythonhosted.org/packages/5a/4a/7bb2eafe4077f6d9867547ca74ca4b75bc8a081e32a47e186e5c067f6cab/cytoolz-1.0.0-cp311-cp311-win32.whl", hash = "sha256:dbb2ec1177dca700f3db2127e572da20de280c214fc587b2a11c717fc421af56", size = 322300 },
    { url = "https://files.pythonhosted.org/packages/5a/ed/a1c955444343224ab1317a41f6575bc640055eb2495e8f9175f8f28bd776/cytoolz-1.0.0-cp311-cp311-win_amd64.whl", hash = "sha256:0983eee73df86e54bb4a79fcc4996aa8b8368fdbf43897f02f9c3bf39c4dc4fb", size = 365539 },
    { url = "https://files.pythonhosted.org/packages/28/2e/a8b71f74ee75f33164bfbc6324ddd1e8d0f425255b1c930141516f51d539/cytoolz-1.0.0-cp312-cp312-macosx_10_13_x86_64.whl", hash = "sha256:10e3986066dc379e30e225b230754d9f5996aa8d84c2accc69c473c21d261e46", size = 414110 },
    { url = "https://files.pythonhosted.org/packages/c6/0a/999af6bb896375b0c687e292a3dcd4edb338a2764bbac40c0ce11eb21c64/cytoolz-1.0.0-cp312-cp312-macosx_11_0_arm64.whl", hash = "sha256:16576f1bb143ee2cb9f719fcc4b845879fb121f9075c7c5e8a5ff4854bd02fc6", size = 390900 },
    { url = "https://files.pythonhosted.org/packages/30/04/02f0ee5339f8c6ef785f06caee85e17e8e0b406e7e553c8fd99a55ff8390/cytoolz-1.0.0-cp312-cp312-manylinux_2_17_aarch64.manylinux2014_aarch64.whl", hash = "sha256:3faa25a1840b984315e8b3ae517312375f4273ffc9a2f035f548b7f916884f37", size = 2090729 },
    { url = "https://files.pythonhosted.org/packages/04/de/296ded5f81ada90ae4db8c06cc34b142cf6c51fabb4c3c78583abba91c36/cytoolz-1.0.0-cp312-cp312-manylinux_2_17_ppc64le.manylinux2014_ppc64le.whl", hash = "sha256:781fce70a277b20fd95dc66811d1a97bb07b611ceea9bda8b7dd3c6a4b05d59a", size = 2155926 },
    { url = "https://files.pythonhosted.org/packages/cc/ce/d5782bdd3d2fd16d87e83e70e14fcfa65ba67ba21cf7e1007505baef7d79/cytoolz-1.0.0-cp312-cp312-manylinux_2_17_s390x.manylinux2014_s390x.whl", hash = "sha256:7a562c25338eb24d419d1e80a7ae12133844ce6fdeb4ab54459daf250088a1b2", size = 2171893 },
    { url = "https://files.pythonhosted.org/packages/d0/02/22a8c74ff13f8a08e8cacd0a0aa34da3a6e3637cf477e376efc61f7567e5/cytoolz-1.0.0-cp312-cp312-manylinux_2_17_x86_64.manylinux2014_x86_64.whl", hash = "sha256:f29d8330aaf070304f7cd5cb7e73e198753624eb0aec278557cccd460c699b5b", size = 2125265 },
    { url = "https://files.pythonhosted.org/packages/50/d1/a3f2e2ced1fa7e2b5607d05ed4de9951491004a4804e96f78778d11bebd4/cytoolz-1.0.0-cp312-cp312-manylinux_2_5_i686.manylinux1_i686.manylinux_2_17_i686.manylinux2014_i686.whl", hash = "sha256:98a96c54aa55ed9c7cdb23c2f0df39a7b4ee518ac54888480b5bdb5ef69c7ef0", size = 1973962 },
    { url = "https://files.pythonhosted.org/packages/3e/10/174d9585e1011824e2e6e79380f8b1c6e49070c35a278d823d996d1c11e6/cytoolz-1.0.0-cp312-cp312-musllinux_1_2_aarch64.whl", hash = "sha256:287d6d7f475882c2ddcbedf8da9a9b37d85b77690779a2d1cdceb5ae3998d52e", size = 2021691 },
    { url = "https://files.pythonhosted.org/packages/84/aa/bebdca3ae140698d3d4fe75ffd4c87a15ee999cee6b994e0831e5a24cdd7/cytoolz-1.0.0-cp312-cp312-musllinux_1_2_i686.whl", hash = "sha256:05a871688df749b982839239fcd3f8ec3b3b4853775d575ff9cd335fa7c75035", size = 2010169 },
    { url = "https://files.pythonhosted.org/packages/2e/9f/8d5940c953534a4d2ae4419bb4fdc1eb5559345fed1f4838708073d7e6b4/cytoolz-1.0.0-cp312-cp312-musllinux_1_2_ppc64le.whl", hash = "sha256:28bb88e1e2f7d6d4b8e0890b06d292c568984d717de3e8381f2ca1dd12af6470", size = 2154314 },
    { url = "https://files.pythonhosted.org/packages/84/bf/a5414601c95afde30a0c038c5d6273c188f1da8ff25a057bd0c683679e5c/cytoolz-1.0.0-cp312-cp312-musllinux_1_2_s390x.whl", hash = "sha256:576a4f1fc73d8836b10458b583f915849da6e4f7914f4ecb623ad95c2508cad5", size = 2188368 },
    { url = "https://files.pythonhosted.org/packages/67/fe/990ea30d56b9b6602f3bf4af77a1bfd9233e6ffb761b11b8864619fed508/cytoolz-1.0.0-cp312-cp312-musllinux_1_2_x86_64.whl", hash = "sha256:509ed3799c47e4ada14f63e41e8f540ac6e2dab97d5d7298934e6abb9d3830ec", size = 2077906 },
    { url = "https://files.pythonhosted.org/packages/ab/95/94b936501e1e23460732631e5d7c6efc4f6c09df21a594dfca9bf30b9411/cytoolz-1.0.0-cp312-cp312-win32.whl", hash = "sha256:9ce25f02b910630f6dc2540dd1e26c9326027ddde6c59f8cab07c56acc70714c", size = 322445 },
    { url = "https://files.pythonhosted.org/packages/be/04/a49b73591b132be5a28c0670229629a3c002cfac59582a1d38b16bdc6fed/cytoolz-1.0.0-cp312-cp312-win_amd64.whl", hash = "sha256:7e53cfcce87e05b7f0ae2fb2b3e5820048cd0bb7b701e92bd8f75c9fbb7c9ae9", size = 364595 },
]

[[package]]
name = "distro"
version = "1.9.0"
source = { registry = "https://pypi.org/simple" }
sdist = { url = "https://files.pythonhosted.org/packages/fc/f8/98eea607f65de6527f8a2e8885fc8015d3e6f5775df186e443e0964a11c3/distro-1.9.0.tar.gz", hash = "sha256:2fa77c6fd8940f116ee1d6b94a2f90b13b5ea8d019b98bc8bafdcabcdd9bdbed", size = 60722 }
wheels = [
    { url = "https://files.pythonhosted.org/packages/12/b3/231ffd4ab1fc9d679809f356cebee130ac7daa00d6d6f3206dd4fd137e9e/distro-1.9.0-py3-none-any.whl", hash = "sha256:7bffd925d65168f85027d8da9af6bddab658135b840670a223589bc0c8ef02b2", size = 20277 },
]

[[package]]
name = "dnspython"
version = "2.7.0"
source = { registry = "https://pypi.org/simple" }
sdist = { url = "https://files.pythonhosted.org/packages/b5/4a/263763cb2ba3816dd94b08ad3a33d5fdae34ecb856678773cc40a3605829/dnspython-2.7.0.tar.gz", hash = "sha256:ce9c432eda0dc91cf618a5cedf1a4e142651196bbcd2c80e89ed5a907e5cfaf1", size = 345197 }
wheels = [
    { url = "https://files.pythonhosted.org/packages/68/1b/e0a87d256e40e8c888847551b20a017a6b98139178505dc7ffb96f04e954/dnspython-2.7.0-py3-none-any.whl", hash = "sha256:b4c34b7d10b51bcc3a5071e7b8dee77939f1e878477eeecc965e9835f63c6c86", size = 313632 },
]

[[package]]
name = "email-validator"
version = "2.2.0"
source = { registry = "https://pypi.org/simple" }
dependencies = [
    { name = "dnspython" },
    { name = "idna" },
]
sdist = { url = "https://files.pythonhosted.org/packages/48/ce/13508a1ec3f8bb981ae4ca79ea40384becc868bfae97fd1c942bb3a001b1/email_validator-2.2.0.tar.gz", hash = "sha256:cb690f344c617a714f22e66ae771445a1ceb46821152df8e165c5f9a364582b7", size = 48967 }
wheels = [
    { url = "https://files.pythonhosted.org/packages/d7/ee/bf0adb559ad3c786f12bcbc9296b3f5675f529199bef03e2df281fa1fadb/email_validator-2.2.0-py3-none-any.whl", hash = "sha256:561977c2d73ce3611850a06fa56b414621e0c8faa9d66f2611407d87465da631", size = 33521 },
]

[[package]]
name = "eth-abi"
version = "5.1.0"
source = { registry = "https://pypi.org/simple" }
dependencies = [
    { name = "eth-typing" },
    { name = "eth-utils" },
    { name = "parsimonious" },
]
sdist = { url = "https://files.pythonhosted.org/packages/91/f7/dc714b95d07ee825f60fc62c26822a5da44b4930d362f8f5ab69eb2d7403/eth_abi-5.1.0.tar.gz", hash = "sha256:33ddd756206e90f7ddff1330cc8cac4aa411a824fe779314a0a52abea2c8fc14", size = 49898 }
wheels = [
    { url = "https://files.pythonhosted.org/packages/42/ec/f30f1cfeea8fd2a4b1b38c9b971014756fcc1cd1136e31dbda66c379b68e/eth_abi-5.1.0-py3-none-any.whl", hash = "sha256:84cac2626a7db8b7d9ebe62b0fdca676ab1014cc7f777189e3c0cd721a4c16d8", size = 29157 },
]

[[package]]
name = "eth-account"
version = "0.13.4"
source = { registry = "https://pypi.org/simple" }
dependencies = [
    { name = "bitarray" },
    { name = "ckzg" },
    { name = "eth-abi" },
    { name = "eth-keyfile" },
    { name = "eth-keys" },
    { name = "eth-rlp" },
    { name = "eth-utils" },
    { name = "hexbytes" },
    { name = "pydantic" },
    { name = "rlp" },
]
sdist = { url = "https://files.pythonhosted.org/packages/fa/c9/38a55928f6eef30c8c7144f00400988bc1950e10b89de09991b315ff9d98/eth_account-0.13.4.tar.gz", hash = "sha256:2e1f2de240bef3d9f3d8013656135d2a79b6be6d4e7885bce9cace4334a4a376", size = 931759 }
wheels = [
    { url = "https://files.pythonhosted.org/packages/03/0b/c8e21c361b61be773e2d9a4a69ec2c72d445a1f1293e026935affe7fc7df/eth_account-0.13.4-py3-none-any.whl", hash = "sha256:a4c109e9bad3a278243fcc028b755fb72b43e25b1e6256b3f309a44f5f7d87c3", size = 581366 },
]

[[package]]
name = "eth-hash"
version = "0.7.0"
source = { registry = "https://pypi.org/simple" }
sdist = { url = "https://files.pythonhosted.org/packages/c6/b6/57c89b91cf2dbb02b3019337f97bf346167d06cd23d3bde43c9fe52cae7e/eth-hash-0.7.0.tar.gz", hash = "sha256:bacdc705bfd85dadd055ecd35fd1b4f846b671add101427e089a4ca2e8db310a", size = 12463 }
wheels = [
    { url = "https://files.pythonhosted.org/packages/73/f0/a35e791bd73fa425838d8d0157754150ded141a94cf30d567dfeb9d57316/eth_hash-0.7.0-py3-none-any.whl", hash = "sha256:b8d5a230a2b251f4a291e3164a23a14057c4a6de4b0aa4a16fa4dc9161b57e2f", size = 8650 },
]

[package.optional-dependencies]
pycryptodome = [
    { name = "pycryptodome" },
]

[[package]]
name = "eth-keyfile"
version = "0.8.1"
source = { registry = "https://pypi.org/simple" }
dependencies = [
    { name = "eth-keys" },
    { name = "eth-utils" },
    { name = "pycryptodome" },
]
sdist = { url = "https://files.pythonhosted.org/packages/35/66/dd823b1537befefbbff602e2ada88f1477c5b40ec3731e3d9bc676c5f716/eth_keyfile-0.8.1.tar.gz", hash = "sha256:9708bc31f386b52cca0969238ff35b1ac72bd7a7186f2a84b86110d3c973bec1", size = 12267 }
wheels = [
    { url = "https://files.pythonhosted.org/packages/88/fc/48a586175f847dd9e05e5b8994d2fe8336098781ec2e9836a2ad94280281/eth_keyfile-0.8.1-py3-none-any.whl", hash = "sha256:65387378b82fe7e86d7cb9f8d98e6d639142661b2f6f490629da09fddbef6d64", size = 7510 },
]

[[package]]
name = "eth-keys"
version = "0.6.0"
source = { registry = "https://pypi.org/simple" }
dependencies = [
    { name = "eth-typing" },
    { name = "eth-utils" },
]
sdist = { url = "https://files.pythonhosted.org/packages/58/4a/aabe0bff4e299858845fba5598c435f2bee0646366b9635750133904e2d8/eth_keys-0.6.0.tar.gz", hash = "sha256:ba33230f851d02c894e83989185b21d76152c49b37e35b61b1d8a6d9f1d20430", size = 28944 }
wheels = [
    { url = "https://files.pythonhosted.org/packages/f0/ee/583612eed5d49f10bd1749d7dda9e93691ab02724b7af84830046e31c64c/eth_keys-0.6.0-py3-none-any.whl", hash = "sha256:b396fdfe048a5bba3ef3990739aec64901eb99901c03921caa774be668b1db6e", size = 21210 },
]

[[package]]
name = "eth-rlp"
version = "2.1.0"
source = { registry = "https://pypi.org/simple" }
dependencies = [
    { name = "eth-utils" },
    { name = "hexbytes" },
    { name = "rlp" },
]
sdist = { url = "https://files.pythonhosted.org/packages/57/4e/aae7407671a1a3cf9c4a1b0c735862112ed7fe409a6753ab2863788702c1/eth-rlp-2.1.0.tar.gz", hash = "sha256:d5b408a8cd20ed496e8e66d0559560d29bc21cee482f893936a1f05d0dddc4a0", size = 7345 }
wheels = [
    { url = "https://files.pythonhosted.org/packages/10/fe/b8b909bf0cc55a8d7d4890bf3a2a37c445a63b194a36f4c6e936fc60a903/eth_rlp-2.1.0-py3-none-any.whl", hash = "sha256:6f476eb7e37d81feaba5d98aed887e467be92648778c44b19fe594aea209cde1", size = 5091 },
]

[[package]]
name = "eth-typing"
version = "5.0.1"
source = { registry = "https://pypi.org/simple" }
dependencies = [
    { name = "typing-extensions" },
]
sdist = { url = "https://files.pythonhosted.org/packages/33/32/d5a1bdf872f92a7c3361396b684aeba7abaabb341bd22a80029abcd1f68e/eth_typing-5.0.1.tar.gz", hash = "sha256:83debf88c9df286db43bb7374974681ebcc9f048fac81be2548dbc549a3203c0", size = 22716 }
wheels = [
    { url = "https://files.pythonhosted.org/packages/1a/ef/a66ff9b83df51b83c1af468fc7b5e4a3855d9e3c01e2365ecfe1c5e84077/eth_typing-5.0.1-py3-none-any.whl", hash = "sha256:f30d1af16aac598f216748a952eeb64fbcb6e73efa691d2de31148138afe96de", size = 20085 },
]

[[package]]
name = "eth-utils"
version = "5.1.0"
source = { registry = "https://pypi.org/simple" }
dependencies = [
    { name = "cytoolz", marker = "implementation_name == 'cpython'" },
    { name = "eth-hash" },
    { name = "eth-typing" },
    { name = "toolz", marker = "implementation_name == 'pypy'" },
]
sdist = { url = "https://files.pythonhosted.org/packages/cc/b2/d5180e5dd01c5f5c9178274c7e27c67579c67554045370d3b8e60b562ea8/eth_utils-5.1.0.tar.gz", hash = "sha256:84c6314b9cf1fcd526107464bbf487e3f87097a2e753360d5ed319f7d42e3f20", size = 120325 }
wheels = [
    { url = "https://files.pythonhosted.org/packages/7c/19/e8d3e0e53dea8fb3054892e536cd1e02ebf492eeb4c42901763c2ae67f44/eth_utils-5.1.0-py3-none-any.whl", hash = "sha256:a99f1f01b51206620904c5af47fac65abc143aebd0a76bdec860381c5a3230f8", size = 100513 },
]

[[package]]
name = "fastapi"
version = "0.111.1"
source = { registry = "https://pypi.org/simple" }
dependencies = [
    { name = "email-validator" },
    { name = "fastapi-cli" },
    { name = "httpx" },
    { name = "jinja2" },
    { name = "pydantic" },
    { name = "python-multipart" },
    { name = "starlette" },
    { name = "typing-extensions" },
    { name = "uvicorn", extra = ["standard"] },
]
sdist = { url = "https://files.pythonhosted.org/packages/a0/ab/9f461ced846964e01c7a6ab25ea79ce2f07743285697b03c8e0e83d72e3e/fastapi-0.111.1.tar.gz", hash = "sha256:ddd1ac34cb1f76c2e2d7f8545a4bcb5463bce4834e81abf0b189e0c359ab2413", size = 288903 }
wheels = [
    { url = "https://files.pythonhosted.org/packages/a4/d4/eb78f7c2648a3585095623f207d7e4b85a1be30347e01e0fdcd1d7d167a9/fastapi-0.111.1-py3-none-any.whl", hash = "sha256:4f51cfa25d72f9fbc3280832e84b32494cf186f50158d364a8765aabf22587bf", size = 92210 },
]

[[package]]
name = "fastapi-cli"
version = "0.0.5"
source = { registry = "https://pypi.org/simple" }
dependencies = [
    { name = "typer" },
    { name = "uvicorn", extra = ["standard"] },
]
sdist = { url = "https://files.pythonhosted.org/packages/c5/f8/1ad5ce32d029aeb9117e9a5a9b3e314a8477525d60c12a9b7730a3c186ec/fastapi_cli-0.0.5.tar.gz", hash = "sha256:d30e1239c6f46fcb95e606f02cdda59a1e2fa778a54b64686b3ff27f6211ff9f", size = 15571 }
wheels = [
    { url = "https://files.pythonhosted.org/packages/24/ea/4b5011012ac925fe2f83b19d0e09cee9d324141ec7bf5e78bb2817f96513/fastapi_cli-0.0.5-py3-none-any.whl", hash = "sha256:e94d847524648c748a5350673546bbf9bcaeb086b33c24f2e82e021436866a46", size = 9489 },
]

[[package]]
name = "frozenlist"
version = "1.5.0"
source = { registry = "https://pypi.org/simple" }
sdist = { url = "https://files.pythonhosted.org/packages/8f/ed/0f4cec13a93c02c47ec32d81d11c0c1efbadf4a471e3f3ce7cad366cbbd3/frozenlist-1.5.0.tar.gz", hash = "sha256:81d5af29e61b9c8348e876d442253723928dce6433e0e76cd925cd83f1b4b817", size = 39930 }
wheels = [
    { url = "https://files.pythonhosted.org/packages/79/43/0bed28bf5eb1c9e4301003b74453b8e7aa85fb293b31dde352aac528dafc/frozenlist-1.5.0-cp311-cp311-macosx_10_9_universal2.whl", hash = "sha256:fd74520371c3c4175142d02a976aee0b4cb4a7cc912a60586ffd8d5929979b30", size = 94987 },
    { url = "https://files.pythonhosted.org/packages/bb/bf/b74e38f09a246e8abbe1e90eb65787ed745ccab6eaa58b9c9308e052323d/frozenlist-1.5.0-cp311-cp311-macosx_10_9_x86_64.whl", hash = "sha256:2f3f7a0fbc219fb4455264cae4d9f01ad41ae6ee8524500f381de64ffaa077d5", size = 54584 },
    { url = "https://files.pythonhosted.org/packages/2c/31/ab01375682f14f7613a1ade30149f684c84f9b8823a4391ed950c8285656/frozenlist-1.5.0-cp311-cp311-macosx_11_0_arm64.whl", hash = "sha256:f47c9c9028f55a04ac254346e92977bf0f166c483c74b4232bee19a6697e4778", size = 52499 },
    { url = "https://files.pythonhosted.org/packages/98/a8/d0ac0b9276e1404f58fec3ab6e90a4f76b778a49373ccaf6a563f100dfbc/frozenlist-1.5.0-cp311-cp311-manylinux_2_17_aarch64.manylinux2014_aarch64.whl", hash = "sha256:0996c66760924da6e88922756d99b47512a71cfd45215f3570bf1e0b694c206a", size = 276357 },
    { url = "https://files.pythonhosted.org/packages/ad/c9/c7761084fa822f07dac38ac29f841d4587570dd211e2262544aa0b791d21/frozenlist-1.5.0-cp311-cp311-manylinux_2_17_ppc64le.manylinux2014_ppc64le.whl", hash = "sha256:a2fe128eb4edeabe11896cb6af88fca5346059f6c8d807e3b910069f39157869", size = 287516 },
    { url = "https://files.pythonhosted.org/packages/a1/ff/cd7479e703c39df7bdab431798cef89dc75010d8aa0ca2514c5b9321db27/frozenlist-1.5.0-cp311-cp311-manylinux_2_17_s390x.manylinux2014_s390x.whl", hash = "sha256:1a8ea951bbb6cacd492e3948b8da8c502a3f814f5d20935aae74b5df2b19cf3d", size = 283131 },
    { url = "https://files.pythonhosted.org/packages/59/a0/370941beb47d237eca4fbf27e4e91389fd68699e6f4b0ebcc95da463835b/frozenlist-1.5.0-cp311-cp311-manylinux_2_5_i686.manylinux1_i686.manylinux_2_17_i686.manylinux2014_i686.whl", hash = "sha256:de537c11e4aa01d37db0d403b57bd6f0546e71a82347a97c6a9f0dcc532b3a45", size = 261320 },
    { url = "https://files.pythonhosted.org/packages/b8/5f/c10123e8d64867bc9b4f2f510a32042a306ff5fcd7e2e09e5ae5100ee333/frozenlist-1.5.0-cp311-cp311-manylinux_2_5_x86_64.manylinux1_x86_64.manylinux_2_17_x86_64.manylinux2014_x86_64.whl", hash = "sha256:9c2623347b933fcb9095841f1cc5d4ff0b278addd743e0e966cb3d460278840d", size = 274877 },
    { url = "https://files.pythonhosted.org/packages/fa/79/38c505601ae29d4348f21706c5d89755ceded02a745016ba2f58bd5f1ea6/frozenlist-1.5.0-cp311-cp311-musllinux_1_2_aarch64.whl", hash = "sha256:cee6798eaf8b1416ef6909b06f7dc04b60755206bddc599f52232606e18179d3", size = 269592 },
    { url = "https://files.pythonhosted.org/packages/19/e2/39f3a53191b8204ba9f0bb574b926b73dd2efba2a2b9d2d730517e8f7622/frozenlist-1.5.0-cp311-cp311-musllinux_1_2_i686.whl", hash = "sha256:f5f9da7f5dbc00a604fe74aa02ae7c98bcede8a3b8b9666f9f86fc13993bc71a", size = 265934 },
    { url = "https://files.pythonhosted.org/packages/d5/c9/3075eb7f7f3a91f1a6b00284af4de0a65a9ae47084930916f5528144c9dd/frozenlist-1.5.0-cp311-cp311-musllinux_1_2_ppc64le.whl", hash = "sha256:90646abbc7a5d5c7c19461d2e3eeb76eb0b204919e6ece342feb6032c9325ae9", size = 283859 },
    { url = "https://files.pythonhosted.org/packages/05/f5/549f44d314c29408b962fa2b0e69a1a67c59379fb143b92a0a065ffd1f0f/frozenlist-1.5.0-cp311-cp311-musllinux_1_2_s390x.whl", hash = "sha256:bdac3c7d9b705d253b2ce370fde941836a5f8b3c5c2b8fd70940a3ea3af7f4f2", size = 287560 },
    { url = "https://files.pythonhosted.org/packages/9d/f8/cb09b3c24a3eac02c4c07a9558e11e9e244fb02bf62c85ac2106d1eb0c0b/frozenlist-1.5.0-cp311-cp311-musllinux_1_2_x86_64.whl", hash = "sha256:03d33c2ddbc1816237a67f66336616416e2bbb6beb306e5f890f2eb22b959cdf", size = 277150 },
    { url = "https://files.pythonhosted.org/packages/37/48/38c2db3f54d1501e692d6fe058f45b6ad1b358d82cd19436efab80cfc965/frozenlist-1.5.0-cp311-cp311-win32.whl", hash = "sha256:237f6b23ee0f44066219dae14c70ae38a63f0440ce6750f868ee08775073f942", size = 45244 },
    { url = "https://files.pythonhosted.org/packages/ca/8c/2ddffeb8b60a4bce3b196c32fcc30d8830d4615e7b492ec2071da801b8ad/frozenlist-1.5.0-cp311-cp311-win_amd64.whl", hash = "sha256:0cc974cc93d32c42e7b0f6cf242a6bd941c57c61b618e78b6c0a96cb72788c1d", size = 51634 },
    { url = "https://files.pythonhosted.org/packages/79/73/fa6d1a96ab7fd6e6d1c3500700963eab46813847f01ef0ccbaa726181dd5/frozenlist-1.5.0-cp312-cp312-macosx_10_13_universal2.whl", hash = "sha256:31115ba75889723431aa9a4e77d5f398f5cf976eea3bdf61749731f62d4a4a21", size = 94026 },
    { url = "https://files.pythonhosted.org/packages/ab/04/ea8bf62c8868b8eada363f20ff1b647cf2e93377a7b284d36062d21d81d1/frozenlist-1.5.0-cp312-cp312-macosx_10_13_x86_64.whl", hash = "sha256:7437601c4d89d070eac8323f121fcf25f88674627505334654fd027b091db09d", size = 54150 },
    { url = "https://files.pythonhosted.org/packages/d0/9a/8e479b482a6f2070b26bda572c5e6889bb3ba48977e81beea35b5ae13ece/frozenlist-1.5.0-cp312-cp312-macosx_11_0_arm64.whl", hash = "sha256:7948140d9f8ece1745be806f2bfdf390127cf1a763b925c4a805c603df5e697e", size = 51927 },
    { url = "https://files.pythonhosted.org/packages/e3/12/2aad87deb08a4e7ccfb33600871bbe8f0e08cb6d8224371387f3303654d7/frozenlist-1.5.0-cp312-cp312-manylinux_2_17_aarch64.manylinux2014_aarch64.whl", hash = "sha256:feeb64bc9bcc6b45c6311c9e9b99406660a9c05ca8a5b30d14a78555088b0b3a", size = 282647 },
    { url = "https://files.pythonhosted.org/packages/77/f2/07f06b05d8a427ea0060a9cef6e63405ea9e0d761846b95ef3fb3be57111/frozenlist-1.5.0-cp312-cp312-manylinux_2_17_ppc64le.manylinux2014_ppc64le.whl", hash = "sha256:683173d371daad49cffb8309779e886e59c2f369430ad28fe715f66d08d4ab1a", size = 289052 },
    { url = "https://files.pythonhosted.org/packages/bd/9f/8bf45a2f1cd4aa401acd271b077989c9267ae8463e7c8b1eb0d3f561b65e/frozenlist-1.5.0-cp312-cp312-manylinux_2_17_s390x.manylinux2014_s390x.whl", hash = "sha256:7d57d8f702221405a9d9b40f9da8ac2e4a1a8b5285aac6100f3393675f0a85ee", size = 291719 },
    { url = "https://files.pythonhosted.org/packages/41/d1/1f20fd05a6c42d3868709b7604c9f15538a29e4f734c694c6bcfc3d3b935/frozenlist-1.5.0-cp312-cp312-manylinux_2_5_i686.manylinux1_i686.manylinux_2_17_i686.manylinux2014_i686.whl", hash = "sha256:30c72000fbcc35b129cb09956836c7d7abf78ab5416595e4857d1cae8d6251a6", size = 267433 },
    { url = "https://files.pythonhosted.org/packages/af/f2/64b73a9bb86f5a89fb55450e97cd5c1f84a862d4ff90d9fd1a73ab0f64a5/frozenlist-1.5.0-cp312-cp312-manylinux_2_5_x86_64.manylinux1_x86_64.manylinux_2_17_x86_64.manylinux2014_x86_64.whl", hash = "sha256:000a77d6034fbad9b6bb880f7ec073027908f1b40254b5d6f26210d2dab1240e", size = 283591 },
    { url = "https://files.pythonhosted.org/packages/29/e2/ffbb1fae55a791fd6c2938dd9ea779509c977435ba3940b9f2e8dc9d5316/frozenlist-1.5.0-cp312-cp312-musllinux_1_2_aarch64.whl", hash = "sha256:5d7f5a50342475962eb18b740f3beecc685a15b52c91f7d975257e13e029eca9", size = 273249 },
    { url = "https://files.pythonhosted.org/packages/2e/6e/008136a30798bb63618a114b9321b5971172a5abddff44a100c7edc5ad4f/frozenlist-1.5.0-cp312-cp312-musllinux_1_2_i686.whl", hash = "sha256:87f724d055eb4785d9be84e9ebf0f24e392ddfad00b3fe036e43f489fafc9039", size = 271075 },
    { url = "https://files.pythonhosted.org/packages/ae/f0/4e71e54a026b06724cec9b6c54f0b13a4e9e298cc8db0f82ec70e151f5ce/frozenlist-1.5.0-cp312-cp312-musllinux_1_2_ppc64le.whl", hash = "sha256:6e9080bb2fb195a046e5177f10d9d82b8a204c0736a97a153c2466127de87784", size = 285398 },
    { url = "https://files.pythonhosted.org/packages/4d/36/70ec246851478b1c0b59f11ef8ade9c482ff447c1363c2bd5fad45098b12/frozenlist-1.5.0-cp312-cp312-musllinux_1_2_s390x.whl", hash = "sha256:9b93d7aaa36c966fa42efcaf716e6b3900438632a626fb09c049f6a2f09fc631", size = 294445 },
    { url = "https://files.pythonhosted.org/packages/37/e0/47f87544055b3349b633a03c4d94b405956cf2437f4ab46d0928b74b7526/frozenlist-1.5.0-cp312-cp312-musllinux_1_2_x86_64.whl", hash = "sha256:52ef692a4bc60a6dd57f507429636c2af8b6046db8b31b18dac02cbc8f507f7f", size = 280569 },
    { url = "https://files.pythonhosted.org/packages/f9/7c/490133c160fb6b84ed374c266f42800e33b50c3bbab1652764e6e1fc498a/frozenlist-1.5.0-cp312-cp312-win32.whl", hash = "sha256:29d94c256679247b33a3dc96cce0f93cbc69c23bf75ff715919332fdbb6a32b8", size = 44721 },
    { url = "https://files.pythonhosted.org/packages/b1/56/4e45136ffc6bdbfa68c29ca56ef53783ef4c2fd395f7cbf99a2624aa9aaa/frozenlist-1.5.0-cp312-cp312-win_amd64.whl", hash = "sha256:8969190d709e7c48ea386db202d708eb94bdb29207a1f269bab1196ce0dcca1f", size = 51329 },
    { url = "https://files.pythonhosted.org/packages/da/3b/915f0bca8a7ea04483622e84a9bd90033bab54bdf485479556c74fd5eaf5/frozenlist-1.5.0-cp313-cp313-macosx_10_13_universal2.whl", hash = "sha256:7a1a048f9215c90973402e26c01d1cff8a209e1f1b53f72b95c13db61b00f953", size = 91538 },
    { url = "https://files.pythonhosted.org/packages/c7/d1/a7c98aad7e44afe5306a2b068434a5830f1470675f0e715abb86eb15f15b/frozenlist-1.5.0-cp313-cp313-macosx_10_13_x86_64.whl", hash = "sha256:dd47a5181ce5fcb463b5d9e17ecfdb02b678cca31280639255ce9d0e5aa67af0", size = 52849 },
    { url = "https://files.pythonhosted.org/packages/3a/c8/76f23bf9ab15d5f760eb48701909645f686f9c64fbb8982674c241fbef14/frozenlist-1.5.0-cp313-cp313-macosx_11_0_arm64.whl", hash = "sha256:1431d60b36d15cda188ea222033eec8e0eab488f39a272461f2e6d9e1a8e63c2", size = 50583 },
    { url = "https://files.pythonhosted.org/packages/1f/22/462a3dd093d11df623179d7754a3b3269de3b42de2808cddef50ee0f4f48/frozenlist-1.5.0-cp313-cp313-manylinux_2_17_aarch64.manylinux2014_aarch64.whl", hash = "sha256:6482a5851f5d72767fbd0e507e80737f9c8646ae7fd303def99bfe813f76cf7f", size = 265636 },
    { url = "https://files.pythonhosted.org/packages/80/cf/e075e407fc2ae7328155a1cd7e22f932773c8073c1fc78016607d19cc3e5/frozenlist-1.5.0-cp313-cp313-manylinux_2_17_ppc64le.manylinux2014_ppc64le.whl", hash = "sha256:44c49271a937625619e862baacbd037a7ef86dd1ee215afc298a417ff3270608", size = 270214 },
    { url = "https://files.pythonhosted.org/packages/a1/58/0642d061d5de779f39c50cbb00df49682832923f3d2ebfb0fedf02d05f7f/frozenlist-1.5.0-cp313-cp313-manylinux_2_17_s390x.manylinux2014_s390x.whl", hash = "sha256:12f78f98c2f1c2429d42e6a485f433722b0061d5c0b0139efa64f396efb5886b", size = 273905 },
    { url = "https://files.pythonhosted.org/packages/ab/66/3fe0f5f8f2add5b4ab7aa4e199f767fd3b55da26e3ca4ce2cc36698e50c4/frozenlist-1.5.0-cp313-cp313-manylinux_2_5_i686.manylinux1_i686.manylinux_2_17_i686.manylinux2014_i686.whl", hash = "sha256:ce3aa154c452d2467487765e3adc730a8c153af77ad84096bc19ce19a2400840", size = 250542 },
    { url = "https://files.pythonhosted.org/packages/f6/b8/260791bde9198c87a465224e0e2bb62c4e716f5d198fc3a1dacc4895dbd1/frozenlist-1.5.0-cp313-cp313-manylinux_2_5_x86_64.manylinux1_x86_64.manylinux_2_17_x86_64.manylinux2014_x86_64.whl", hash = "sha256:9b7dc0c4338e6b8b091e8faf0db3168a37101943e687f373dce00959583f7439", size = 267026 },
    { url = "https://files.pythonhosted.org/packages/2e/a4/3d24f88c527f08f8d44ade24eaee83b2627793fa62fa07cbb7ff7a2f7d42/frozenlist-1.5.0-cp313-cp313-musllinux_1_2_aarch64.whl", hash = "sha256:45e0896250900b5aa25180f9aec243e84e92ac84bd4a74d9ad4138ef3f5c97de", size = 257690 },
    { url = "https://files.pythonhosted.org/packages/de/9a/d311d660420b2beeff3459b6626f2ab4fb236d07afbdac034a4371fe696e/frozenlist-1.5.0-cp313-cp313-musllinux_1_2_i686.whl", hash = "sha256:561eb1c9579d495fddb6da8959fd2a1fca2c6d060d4113f5844b433fc02f2641", size = 253893 },
    { url = "https://files.pythonhosted.org/packages/c6/23/e491aadc25b56eabd0f18c53bb19f3cdc6de30b2129ee0bc39cd387cd560/frozenlist-1.5.0-cp313-cp313-musllinux_1_2_ppc64le.whl", hash = "sha256:df6e2f325bfee1f49f81aaac97d2aa757c7646534a06f8f577ce184afe2f0a9e", size = 267006 },
    { url = "https://files.pythonhosted.org/packages/08/c4/ab918ce636a35fb974d13d666dcbe03969592aeca6c3ab3835acff01f79c/frozenlist-1.5.0-cp313-cp313-musllinux_1_2_s390x.whl", hash = "sha256:140228863501b44b809fb39ec56b5d4071f4d0aa6d216c19cbb08b8c5a7eadb9", size = 276157 },
    { url = "https://files.pythonhosted.org/packages/c0/29/3b7a0bbbbe5a34833ba26f686aabfe982924adbdcafdc294a7a129c31688/frozenlist-1.5.0-cp313-cp313-musllinux_1_2_x86_64.whl", hash = "sha256:7707a25d6a77f5d27ea7dc7d1fc608aa0a478193823f88511ef5e6b8a48f9d03", size = 264642 },
    { url = "https://files.pythonhosted.org/packages/ab/42/0595b3dbffc2e82d7fe658c12d5a5bafcd7516c6bf2d1d1feb5387caa9c1/frozenlist-1.5.0-cp313-cp313-win32.whl", hash = "sha256:31a9ac2b38ab9b5a8933b693db4939764ad3f299fcaa931a3e605bc3460e693c", size = 44914 },
    { url = "https://files.pythonhosted.org/packages/17/c4/b7db1206a3fea44bf3b838ca61deb6f74424a8a5db1dd53ecb21da669be6/frozenlist-1.5.0-cp313-cp313-win_amd64.whl", hash = "sha256:11aabdd62b8b9c4b84081a3c246506d1cddd2dd93ff0ad53ede5defec7886b28", size = 51167 },
    { url = "https://files.pythonhosted.org/packages/c6/c8/a5be5b7550c10858fcf9b0ea054baccab474da77d37f1e828ce043a3a5d4/frozenlist-1.5.0-py3-none-any.whl", hash = "sha256:d994863bba198a4a518b467bb971c56e1db3f180a25c6cf7bb1949c267f748c3", size = 11901 },
]

[[package]]
name = "greenlet"
version = "3.1.1"
source = { registry = "https://pypi.org/simple" }
sdist = { url = "https://files.pythonhosted.org/packages/2f/ff/df5fede753cc10f6a5be0931204ea30c35fa2f2ea7a35b25bdaf4fe40e46/greenlet-3.1.1.tar.gz", hash = "sha256:4ce3ac6cdb6adf7946475d7ef31777c26d94bccc377e070a7986bd2d5c515467", size = 186022 }
wheels = [
    { url = "https://files.pythonhosted.org/packages/28/62/1c2665558618553c42922ed47a4e6d6527e2fa3516a8256c2f431c5d0441/greenlet-3.1.1-cp311-cp311-macosx_11_0_universal2.whl", hash = "sha256:e4d333e558953648ca09d64f13e6d8f0523fa705f51cae3f03b5983489958c70", size = 272479 },
    { url = "https://files.pythonhosted.org/packages/76/9d/421e2d5f07285b6e4e3a676b016ca781f63cfe4a0cd8eaecf3fd6f7a71ae/greenlet-3.1.1-cp311-cp311-manylinux_2_17_aarch64.manylinux2014_aarch64.whl", hash = "sha256:09fc016b73c94e98e29af67ab7b9a879c307c6731a2c9da0db5a7d9b7edd1159", size = 640404 },
    { url = "https://files.pythonhosted.org/packages/e5/de/6e05f5c59262a584e502dd3d261bbdd2c97ab5416cc9c0b91ea38932a901/greenlet-3.1.1-cp311-cp311-manylinux_2_17_ppc64le.manylinux2014_ppc64le.whl", hash = "sha256:d5e975ca70269d66d17dd995dafc06f1b06e8cb1ec1e9ed54c1d1e4a7c4cf26e", size = 652813 },
    { url = "https://files.pythonhosted.org/packages/49/93/d5f93c84241acdea15a8fd329362c2c71c79e1a507c3f142a5d67ea435ae/greenlet-3.1.1-cp311-cp311-manylinux_2_17_s390x.manylinux2014_s390x.whl", hash = "sha256:3b2813dc3de8c1ee3f924e4d4227999285fd335d1bcc0d2be6dc3f1f6a318ec1", size = 648517 },
    { url = "https://files.pythonhosted.org/packages/15/85/72f77fc02d00470c86a5c982b8daafdf65d38aefbbe441cebff3bf7037fc/greenlet-3.1.1-cp311-cp311-manylinux_2_17_x86_64.manylinux2014_x86_64.whl", hash = "sha256:e347b3bfcf985a05e8c0b7d462ba6f15b1ee1c909e2dcad795e49e91b152c383", size = 647831 },
    { url = "https://files.pythonhosted.org/packages/f7/4b/1c9695aa24f808e156c8f4813f685d975ca73c000c2a5056c514c64980f6/greenlet-3.1.1-cp311-cp311-manylinux_2_24_x86_64.manylinux_2_28_x86_64.whl", hash = "sha256:9e8f8c9cb53cdac7ba9793c276acd90168f416b9ce36799b9b885790f8ad6c0a", size = 602413 },
    { url = "https://files.pythonhosted.org/packages/76/70/ad6e5b31ef330f03b12559d19fda2606a522d3849cde46b24f223d6d1619/greenlet-3.1.1-cp311-cp311-musllinux_1_1_aarch64.whl", hash = "sha256:62ee94988d6b4722ce0028644418d93a52429e977d742ca2ccbe1c4f4a792511", size = 1129619 },
    { url = "https://files.pythonhosted.org/packages/f4/fb/201e1b932e584066e0f0658b538e73c459b34d44b4bd4034f682423bc801/greenlet-3.1.1-cp311-cp311-musllinux_1_1_x86_64.whl", hash = "sha256:1776fd7f989fc6b8d8c8cb8da1f6b82c5814957264d1f6cf818d475ec2bf6395", size = 1155198 },
    { url = "https://files.pythonhosted.org/packages/12/da/b9ed5e310bb8b89661b80cbcd4db5a067903bbcd7fc854923f5ebb4144f0/greenlet-3.1.1-cp311-cp311-win_amd64.whl", hash = "sha256:48ca08c771c268a768087b408658e216133aecd835c0ded47ce955381105ba39", size = 298930 },
    { url = "https://files.pythonhosted.org/packages/7d/ec/bad1ac26764d26aa1353216fcbfa4670050f66d445448aafa227f8b16e80/greenlet-3.1.1-cp312-cp312-macosx_11_0_universal2.whl", hash = "sha256:4afe7ea89de619adc868e087b4d2359282058479d7cfb94970adf4b55284574d", size = 274260 },
    { url = "https://files.pythonhosted.org/packages/66/d4/c8c04958870f482459ab5956c2942c4ec35cac7fe245527f1039837c17a9/greenlet-3.1.1-cp312-cp312-manylinux_2_17_aarch64.manylinux2014_aarch64.whl", hash = "sha256:f406b22b7c9a9b4f8aa9d2ab13d6ae0ac3e85c9a809bd590ad53fed2bf70dc79", size = 649064 },
    { url = "https://files.pythonhosted.org/packages/51/41/467b12a8c7c1303d20abcca145db2be4e6cd50a951fa30af48b6ec607581/greenlet-3.1.1-cp312-cp312-manylinux_2_17_ppc64le.manylinux2014_ppc64le.whl", hash = "sha256:c3a701fe5a9695b238503ce5bbe8218e03c3bcccf7e204e455e7462d770268aa", size = 663420 },
    { url = "https://files.pythonhosted.org/packages/27/8f/2a93cd9b1e7107d5c7b3b7816eeadcac2ebcaf6d6513df9abaf0334777f6/greenlet-3.1.1-cp312-cp312-manylinux_2_17_s390x.manylinux2014_s390x.whl", hash = "sha256:2846930c65b47d70b9d178e89c7e1a69c95c1f68ea5aa0a58646b7a96df12441", size = 658035 },
    { url = "https://files.pythonhosted.org/packages/57/5c/7c6f50cb12be092e1dccb2599be5a942c3416dbcfb76efcf54b3f8be4d8d/greenlet-3.1.1-cp312-cp312-manylinux_2_17_x86_64.manylinux2014_x86_64.whl", hash = "sha256:99cfaa2110534e2cf3ba31a7abcac9d328d1d9f1b95beede58294a60348fba36", size = 660105 },
    { url = "https://files.pythonhosted.org/packages/f1/66/033e58a50fd9ec9df00a8671c74f1f3a320564c6415a4ed82a1c651654ba/greenlet-3.1.1-cp312-cp312-manylinux_2_24_x86_64.manylinux_2_28_x86_64.whl", hash = "sha256:1443279c19fca463fc33e65ef2a935a5b09bb90f978beab37729e1c3c6c25fe9", size = 613077 },
    { url = "https://files.pythonhosted.org/packages/19/c5/36384a06f748044d06bdd8776e231fadf92fc896bd12cb1c9f5a1bda9578/greenlet-3.1.1-cp312-cp312-musllinux_1_1_aarch64.whl", hash = "sha256:b7cede291382a78f7bb5f04a529cb18e068dd29e0fb27376074b6d0317bf4dd0", size = 1135975 },
    { url = "https://files.pythonhosted.org/packages/38/f9/c0a0eb61bdf808d23266ecf1d63309f0e1471f284300ce6dac0ae1231881/greenlet-3.1.1-cp312-cp312-musllinux_1_1_x86_64.whl", hash = "sha256:23f20bb60ae298d7d8656c6ec6db134bca379ecefadb0b19ce6f19d1f232a942", size = 1163955 },
    { url = "https://files.pythonhosted.org/packages/43/21/a5d9df1d21514883333fc86584c07c2b49ba7c602e670b174bd73cfc9c7f/greenlet-3.1.1-cp312-cp312-win_amd64.whl", hash = "sha256:7124e16b4c55d417577c2077be379514321916d5790fa287c9ed6f23bd2ffd01", size = 299655 },
    { url = "https://files.pythonhosted.org/packages/f3/57/0db4940cd7bb461365ca8d6fd53e68254c9dbbcc2b452e69d0d41f10a85e/greenlet-3.1.1-cp313-cp313-macosx_11_0_universal2.whl", hash = "sha256:05175c27cb459dcfc05d026c4232f9de8913ed006d42713cb8a5137bd49375f1", size = 272990 },
    { url = "https://files.pythonhosted.org/packages/1c/ec/423d113c9f74e5e402e175b157203e9102feeb7088cee844d735b28ef963/greenlet-3.1.1-cp313-cp313-manylinux_2_17_aarch64.manylinux2014_aarch64.whl", hash = "sha256:935e943ec47c4afab8965954bf49bfa639c05d4ccf9ef6e924188f762145c0ff", size = 649175 },
    { url = "https://files.pythonhosted.org/packages/a9/46/ddbd2db9ff209186b7b7c621d1432e2f21714adc988703dbdd0e65155c77/greenlet-3.1.1-cp313-cp313-manylinux_2_17_ppc64le.manylinux2014_ppc64le.whl", hash = "sha256:667a9706c970cb552ede35aee17339a18e8f2a87a51fba2ed39ceeeb1004798a", size = 663425 },
    { url = "https://files.pythonhosted.org/packages/bc/f9/9c82d6b2b04aa37e38e74f0c429aece5eeb02bab6e3b98e7db89b23d94c6/greenlet-3.1.1-cp313-cp313-manylinux_2_17_s390x.manylinux2014_s390x.whl", hash = "sha256:b8a678974d1f3aa55f6cc34dc480169d58f2e6d8958895d68845fa4ab566509e", size = 657736 },
    { url = "https://files.pythonhosted.org/packages/d9/42/b87bc2a81e3a62c3de2b0d550bf91a86939442b7ff85abb94eec3fc0e6aa/greenlet-3.1.1-cp313-cp313-manylinux_2_17_x86_64.manylinux2014_x86_64.whl", hash = "sha256:efc0f674aa41b92da8c49e0346318c6075d734994c3c4e4430b1c3f853e498e4", size = 660347 },
    { url = "https://files.pythonhosted.org/packages/37/fa/71599c3fd06336cdc3eac52e6871cfebab4d9d70674a9a9e7a482c318e99/greenlet-3.1.1-cp313-cp313-manylinux_2_24_x86_64.manylinux_2_28_x86_64.whl", hash = "sha256:0153404a4bb921f0ff1abeb5ce8a5131da56b953eda6e14b88dc6bbc04d2049e", size = 615583 },
    { url = "https://files.pythonhosted.org/packages/4e/96/e9ef85de031703ee7a4483489b40cf307f93c1824a02e903106f2ea315fe/greenlet-3.1.1-cp313-cp313-musllinux_1_1_aarch64.whl", hash = "sha256:275f72decf9932639c1c6dd1013a1bc266438eb32710016a1c742df5da6e60a1", size = 1133039 },
    { url = "https://files.pythonhosted.org/packages/87/76/b2b6362accd69f2d1889db61a18c94bc743e961e3cab344c2effaa4b4a25/greenlet-3.1.1-cp313-cp313-musllinux_1_1_x86_64.whl", hash = "sha256:c4aab7f6381f38a4b42f269057aee279ab0fc7bf2e929e3d4abfae97b682a12c", size = 1160716 },
    { url = "https://files.pythonhosted.org/packages/1f/1b/54336d876186920e185066d8c3024ad55f21d7cc3683c856127ddb7b13ce/greenlet-3.1.1-cp313-cp313-win_amd64.whl", hash = "sha256:b42703b1cf69f2aa1df7d1030b9d77d3e584a70755674d60e710f0af570f3761", size = 299490 },
    { url = "https://files.pythonhosted.org/packages/5f/17/bea55bf36990e1638a2af5ba10c1640273ef20f627962cf97107f1e5d637/greenlet-3.1.1-cp313-cp313t-manylinux_2_17_aarch64.manylinux2014_aarch64.whl", hash = "sha256:f1695e76146579f8c06c1509c7ce4dfe0706f49c6831a817ac04eebb2fd02011", size = 643731 },
    { url = "https://files.pythonhosted.org/packages/78/d2/aa3d2157f9ab742a08e0fd8f77d4699f37c22adfbfeb0c610a186b5f75e0/greenlet-3.1.1-cp313-cp313t-manylinux_2_17_ppc64le.manylinux2014_ppc64le.whl", hash = "sha256:7876452af029456b3f3549b696bb36a06db7c90747740c5302f74a9e9fa14b13", size = 649304 },
    { url = "https://files.pythonhosted.org/packages/f1/8e/d0aeffe69e53ccff5a28fa86f07ad1d2d2d6537a9506229431a2a02e2f15/greenlet-3.1.1-cp313-cp313t-manylinux_2_17_s390x.manylinux2014_s390x.whl", hash = "sha256:4ead44c85f8ab905852d3de8d86f6f8baf77109f9da589cb4fa142bd3b57b475", size = 646537 },
    { url = "https://files.pythonhosted.org/packages/05/79/e15408220bbb989469c8871062c97c6c9136770657ba779711b90870d867/greenlet-3.1.1-cp313-cp313t-manylinux_2_17_x86_64.manylinux2014_x86_64.whl", hash = "sha256:8320f64b777d00dd7ccdade271eaf0cad6636343293a25074cc5566160e4de7b", size = 642506 },
    { url = "https://files.pythonhosted.org/packages/18/87/470e01a940307796f1d25f8167b551a968540fbe0551c0ebb853cb527dd6/greenlet-3.1.1-cp313-cp313t-manylinux_2_24_x86_64.manylinux_2_28_x86_64.whl", hash = "sha256:6510bf84a6b643dabba74d3049ead221257603a253d0a9873f55f6a59a65f822", size = 602753 },
    { url = "https://files.pythonhosted.org/packages/e2/72/576815ba674eddc3c25028238f74d7b8068902b3968cbe456771b166455e/greenlet-3.1.1-cp313-cp313t-musllinux_1_1_aarch64.whl", hash = "sha256:04b013dc07c96f83134b1e99888e7a79979f1a247e2a9f59697fa14b5862ed01", size = 1122731 },
    { url = "https://files.pythonhosted.org/packages/ac/38/08cc303ddddc4b3d7c628c3039a61a3aae36c241ed01393d00c2fd663473/greenlet-3.1.1-cp313-cp313t-musllinux_1_1_x86_64.whl", hash = "sha256:411f015496fec93c1c8cd4e5238da364e1da7a124bcb293f085bf2860c32c6f6", size = 1142112 },
]

[[package]]
name = "h11"
version = "0.14.0"
source = { registry = "https://pypi.org/simple" }
sdist = { url = "https://files.pythonhosted.org/packages/f5/38/3af3d3633a34a3316095b39c8e8fb4853a28a536e55d347bd8d8e9a14b03/h11-0.14.0.tar.gz", hash = "sha256:8f19fbbe99e72420ff35c00b27a34cb9937e902a8b810e2c88300c6f0a3b699d", size = 100418 }
wheels = [
    { url = "https://files.pythonhosted.org/packages/95/04/ff642e65ad6b90db43e668d70ffb6736436c7ce41fcc549f4e9472234127/h11-0.14.0-py3-none-any.whl", hash = "sha256:e3fe4ac4b851c468cc8363d500db52c2ead036020723024a109d37346efaa761", size = 58259 },
]

[[package]]
name = "hexbytes"
version = "1.2.1"
source = { registry = "https://pypi.org/simple" }
sdist = { url = "https://files.pythonhosted.org/packages/4e/51/06836a542b773bfc60ab871fa08d4a7963e7df6754ca57169e2654287cc1/hexbytes-1.2.1.tar.gz", hash = "sha256:515f00dddf31053db4d0d7636dd16061c1d896c3109b8e751005db4ca46bcca7", size = 7722 }
wheels = [
    { url = "https://files.pythonhosted.org/packages/39/c6/20f25ea73e4ceffb3eb4e38347f2992cb25e5ff6eb644d52e753a7a72f57/hexbytes-1.2.1-py3-none-any.whl", hash = "sha256:e64890b203a31f4a23ef11470ecfcca565beaee9198df623047df322b757471a", size = 5160 },
]

[[package]]
name = "httpcore"
version = "1.0.6"
source = { registry = "https://pypi.org/simple" }
dependencies = [
    { name = "certifi" },
    { name = "h11" },
]
sdist = { url = "https://files.pythonhosted.org/packages/b6/44/ed0fa6a17845fb033bd885c03e842f08c1b9406c86a2e60ac1ae1b9206a6/httpcore-1.0.6.tar.gz", hash = "sha256:73f6dbd6eb8c21bbf7ef8efad555481853f5f6acdeaff1edb0694289269ee17f", size = 85180 }
wheels = [
    { url = "https://files.pythonhosted.org/packages/06/89/b161908e2f51be56568184aeb4a880fd287178d176fd1c860d2217f41106/httpcore-1.0.6-py3-none-any.whl", hash = "sha256:27b59625743b85577a8c0e10e55b50b5368a4f2cfe8cc7bcfa9cf00829c2682f", size = 78011 },
]

[[package]]
name = "httptools"
version = "0.6.4"
source = { registry = "https://pypi.org/simple" }
sdist = { url = "https://files.pythonhosted.org/packages/a7/9a/ce5e1f7e131522e6d3426e8e7a490b3a01f39a6696602e1c4f33f9e94277/httptools-0.6.4.tar.gz", hash = "sha256:4e93eee4add6493b59a5c514da98c939b244fce4a0d8879cd3f466562f4b7d5c", size = 240639 }
wheels = [
    { url = "https://files.pythonhosted.org/packages/7b/26/bb526d4d14c2774fe07113ca1db7255737ffbb119315839af2065abfdac3/httptools-0.6.4-cp311-cp311-macosx_10_9_universal2.whl", hash = "sha256:f47f8ed67cc0ff862b84a1189831d1d33c963fb3ce1ee0c65d3b0cbe7b711069", size = 199029 },
    { url = "https://files.pythonhosted.org/packages/a6/17/3e0d3e9b901c732987a45f4f94d4e2c62b89a041d93db89eafb262afd8d5/httptools-0.6.4-cp311-cp311-macosx_11_0_arm64.whl", hash = "sha256:0614154d5454c21b6410fdf5262b4a3ddb0f53f1e1721cfd59d55f32138c578a", size = 103492 },
    { url = "https://files.pythonhosted.org/packages/b7/24/0fe235d7b69c42423c7698d086d4db96475f9b50b6ad26a718ef27a0bce6/httptools-0.6.4-cp311-cp311-manylinux_2_17_aarch64.manylinux2014_aarch64.whl", hash = "sha256:f8787367fbdfccae38e35abf7641dafc5310310a5987b689f4c32cc8cc3ee975", size = 462891 },
    { url = "https://files.pythonhosted.org/packages/b1/2f/205d1f2a190b72da6ffb5f41a3736c26d6fa7871101212b15e9b5cd8f61d/httptools-0.6.4-cp311-cp311-manylinux_2_5_x86_64.manylinux1_x86_64.manylinux_2_17_x86_64.manylinux2014_x86_64.whl", hash = "sha256:40b0f7fe4fd38e6a507bdb751db0379df1e99120c65fbdc8ee6c1d044897a636", size = 459788 },
    { url = "https://files.pythonhosted.org/packages/6e/4c/d09ce0eff09057a206a74575ae8f1e1e2f0364d20e2442224f9e6612c8b9/httptools-0.6.4-cp311-cp311-musllinux_1_2_aarch64.whl", hash = "sha256:40a5ec98d3f49904b9fe36827dcf1aadfef3b89e2bd05b0e35e94f97c2b14721", size = 433214 },
    { url = "https://files.pythonhosted.org/packages/3e/d2/84c9e23edbccc4a4c6f96a1b8d99dfd2350289e94f00e9ccc7aadde26fb5/httptools-0.6.4-cp311-cp311-musllinux_1_2_x86_64.whl", hash = "sha256:dacdd3d10ea1b4ca9df97a0a303cbacafc04b5cd375fa98732678151643d4988", size = 434120 },
    { url = "https://files.pythonhosted.org/packages/d0/46/4d8e7ba9581416de1c425b8264e2cadd201eb709ec1584c381f3e98f51c1/httptools-0.6.4-cp311-cp311-win_amd64.whl", hash = "sha256:288cd628406cc53f9a541cfaf06041b4c71d751856bab45e3702191f931ccd17", size = 88565 },
    { url = "https://files.pythonhosted.org/packages/bb/0e/d0b71465c66b9185f90a091ab36389a7352985fe857e352801c39d6127c8/httptools-0.6.4-cp312-cp312-macosx_10_13_universal2.whl", hash = "sha256:df017d6c780287d5c80601dafa31f17bddb170232d85c066604d8558683711a2", size = 200683 },
    { url = "https://files.pythonhosted.org/packages/e2/b8/412a9bb28d0a8988de3296e01efa0bd62068b33856cdda47fe1b5e890954/httptools-0.6.4-cp312-cp312-macosx_11_0_arm64.whl", hash = "sha256:85071a1e8c2d051b507161f6c3e26155b5c790e4e28d7f236422dbacc2a9cc44", size = 104337 },
    { url = "https://files.pythonhosted.org/packages/9b/01/6fb20be3196ffdc8eeec4e653bc2a275eca7f36634c86302242c4fbb2760/httptools-0.6.4-cp312-cp312-manylinux_2_17_aarch64.manylinux2014_aarch64.whl", hash = "sha256:69422b7f458c5af875922cdb5bd586cc1f1033295aa9ff63ee196a87519ac8e1", size = 508796 },
    { url = "https://files.pythonhosted.org/packages/f7/d8/b644c44acc1368938317d76ac991c9bba1166311880bcc0ac297cb9d6bd7/httptools-0.6.4-cp312-cp312-manylinux_2_5_x86_64.manylinux1_x86_64.manylinux_2_17_x86_64.manylinux2014_x86_64.whl", hash = "sha256:16e603a3bff50db08cd578d54f07032ca1631450ceb972c2f834c2b860c28ea2", size = 510837 },
    { url = "https://files.pythonhosted.org/packages/52/d8/254d16a31d543073a0e57f1c329ca7378d8924e7e292eda72d0064987486/httptools-0.6.4-cp312-cp312-musllinux_1_2_aarch64.whl", hash = "sha256:ec4f178901fa1834d4a060320d2f3abc5c9e39766953d038f1458cb885f47e81", size = 485289 },
    { url = "https://files.pythonhosted.org/packages/5f/3c/4aee161b4b7a971660b8be71a92c24d6c64372c1ab3ae7f366b3680df20f/httptools-0.6.4-cp312-cp312-musllinux_1_2_x86_64.whl", hash = "sha256:f9eb89ecf8b290f2e293325c646a211ff1c2493222798bb80a530c5e7502494f", size = 489779 },
    { url = "https://files.pythonhosted.org/packages/12/b7/5cae71a8868e555f3f67a50ee7f673ce36eac970f029c0c5e9d584352961/httptools-0.6.4-cp312-cp312-win_amd64.whl", hash = "sha256:db78cb9ca56b59b016e64b6031eda5653be0589dba2b1b43453f6e8b405a0970", size = 88634 },
    { url = "https://files.pythonhosted.org/packages/94/a3/9fe9ad23fd35f7de6b91eeb60848986058bd8b5a5c1e256f5860a160cc3e/httptools-0.6.4-cp313-cp313-macosx_10_13_universal2.whl", hash = "sha256:ade273d7e767d5fae13fa637f4d53b6e961fb7fd93c7797562663f0171c26660", size = 197214 },
    { url = "https://files.pythonhosted.org/packages/ea/d9/82d5e68bab783b632023f2fa31db20bebb4e89dfc4d2293945fd68484ee4/httptools-0.6.4-cp313-cp313-macosx_11_0_arm64.whl", hash = "sha256:856f4bc0478ae143bad54a4242fccb1f3f86a6e1be5548fecfd4102061b3a083", size = 102431 },
    { url = "https://files.pythonhosted.org/packages/96/c1/cb499655cbdbfb57b577734fde02f6fa0bbc3fe9fb4d87b742b512908dff/httptools-0.6.4-cp313-cp313-manylinux_2_17_aarch64.manylinux2014_aarch64.whl", hash = "sha256:322d20ea9cdd1fa98bd6a74b77e2ec5b818abdc3d36695ab402a0de8ef2865a3", size = 473121 },
    { url = "https://files.pythonhosted.org/packages/af/71/ee32fd358f8a3bb199b03261f10921716990808a675d8160b5383487a317/httptools-0.6.4-cp313-cp313-manylinux_2_5_x86_64.manylinux1_x86_64.manylinux_2_17_x86_64.manylinux2014_x86_64.whl", hash = "sha256:4d87b29bd4486c0093fc64dea80231f7c7f7eb4dc70ae394d70a495ab8436071", size = 473805 },
    { url = "https://files.pythonhosted.org/packages/8a/0a/0d4df132bfca1507114198b766f1737d57580c9ad1cf93c1ff673e3387be/httptools-0.6.4-cp313-cp313-musllinux_1_2_aarch64.whl", hash = "sha256:342dd6946aa6bda4b8f18c734576106b8a31f2fe31492881a9a160ec84ff4bd5", size = 448858 },
    { url = "https://files.pythonhosted.org/packages/1e/6a/787004fdef2cabea27bad1073bf6a33f2437b4dbd3b6fb4a9d71172b1c7c/httptools-0.6.4-cp313-cp313-musllinux_1_2_x86_64.whl", hash = "sha256:4b36913ba52008249223042dca46e69967985fb4051951f94357ea681e1f5dc0", size = 452042 },
    { url = "https://files.pythonhosted.org/packages/4d/dc/7decab5c404d1d2cdc1bb330b1bf70e83d6af0396fd4fc76fc60c0d522bf/httptools-0.6.4-cp313-cp313-win_amd64.whl", hash = "sha256:28908df1b9bb8187393d5b5db91435ccc9c8e891657f9cbb42a2541b44c82fc8", size = 87682 },
]

[[package]]
name = "httpx"
version = "0.27.2"
source = { registry = "https://pypi.org/simple" }
dependencies = [
    { name = "anyio" },
    { name = "certifi" },
    { name = "httpcore" },
    { name = "idna" },
    { name = "sniffio" },
]
sdist = { url = "https://files.pythonhosted.org/packages/78/82/08f8c936781f67d9e6b9eeb8a0c8b4e406136ea4c3d1f89a5db71d42e0e6/httpx-0.27.2.tar.gz", hash = "sha256:f7c2be1d2f3c3c3160d441802406b206c2b76f5947b11115e6df10c6c65e66c2", size = 144189 }
wheels = [
    { url = "https://files.pythonhosted.org/packages/56/95/9377bcb415797e44274b51d46e3249eba641711cf3348050f76ee7b15ffc/httpx-0.27.2-py3-none-any.whl", hash = "sha256:7bb2708e112d8fdd7829cd4243970f0c223274051cb35ee80c03301ee29a3df0", size = 76395 },
]

[[package]]
name = "idna"
version = "3.10"
source = { registry = "https://pypi.org/simple" }
sdist = { url = "https://files.pythonhosted.org/packages/f1/70/7703c29685631f5a7590aa73f1f1d3fa9a380e654b86af429e0934a32f7d/idna-3.10.tar.gz", hash = "sha256:12f65c9b470abda6dc35cf8e63cc574b1c52b11df2c86030af0ac09b01b13ea9", size = 190490 }
wheels = [
    { url = "https://files.pythonhosted.org/packages/76/c6/c88e154df9c4e1a2a66ccf0005a88dfb2650c1dffb6f5ce603dfbd452ce3/idna-3.10-py3-none-any.whl", hash = "sha256:946d195a0d259cbba61165e88e65941f16e9b36ea6ddb97f00452bae8b1287d3", size = 70442 },
]

[[package]]
name = "jinja2"
version = "3.1.4"
source = { registry = "https://pypi.org/simple" }
dependencies = [
    { name = "markupsafe" },
]
sdist = { url = "https://files.pythonhosted.org/packages/ed/55/39036716d19cab0747a5020fc7e907f362fbf48c984b14e62127f7e68e5d/jinja2-3.1.4.tar.gz", hash = "sha256:4a3aee7acbbe7303aede8e9648d13b8bf88a429282aa6122a993f0ac800cb369", size = 240245 }
wheels = [
    { url = "https://files.pythonhosted.org/packages/31/80/3a54838c3fb461f6fec263ebf3a3a41771bd05190238de3486aae8540c36/jinja2-3.1.4-py3-none-any.whl", hash = "sha256:bc5dd2abb727a5319567b7a813e6a2e7318c39f4f487cfe6c89c6f9c7d25197d", size = 133271 },
]

[[package]]
name = "jiter"
version = "0.6.1"
source = { registry = "https://pypi.org/simple" }
sdist = { url = "https://files.pythonhosted.org/packages/26/ef/64458dfad180debd70d9dd1ca4f607e52bb6de748e5284d748556a0d5173/jiter-0.6.1.tar.gz", hash = "sha256:e19cd21221fc139fb032e4112986656cb2739e9fe6d84c13956ab30ccc7d4449", size = 161306 }
wheels = [
    { url = "https://files.pythonhosted.org/packages/95/91/d1605f3cabcf47193ecab3712e5a4c55a19cf1a4d86ef67402325e28a44e/jiter-0.6.1-cp311-cp311-macosx_10_12_x86_64.whl", hash = "sha256:b03c24e7da7e75b170c7b2b172d9c5e463aa4b5c95696a368d52c295b3f6847f", size = 290963 },
    { url = "https://files.pythonhosted.org/packages/91/35/85ef9eaef7dec14f28dd9b8a2116c07075bb2731a405b650a55fda4c74d7/jiter-0.6.1-cp311-cp311-macosx_11_0_arm64.whl", hash = "sha256:47fee1be677b25d0ef79d687e238dc6ac91a8e553e1a68d0839f38c69e0ee491", size = 302639 },
    { url = "https://files.pythonhosted.org/packages/3b/c7/87a809bf95eb6fbcd8b30ea1d0f922c2187590de64a7f0944615008fde45/jiter-0.6.1-cp311-cp311-manylinux_2_17_aarch64.manylinux2014_aarch64.whl", hash = "sha256:25f0d2f6e01a8a0fb0eab6d0e469058dab2be46ff3139ed2d1543475b5a1d8e7", size = 337048 },
    { url = "https://files.pythonhosted.org/packages/bf/70/c31f21c109a01e6ebb0e032c8296d24761b5244b37d16bb3e9b0789a0eb0/jiter-0.6.1-cp311-cp311-manylinux_2_17_armv7l.manylinux2014_armv7l.whl", hash = "sha256:0b809e39e342c346df454b29bfcc7bca3d957f5d7b60e33dae42b0e5ec13e027", size = 354239 },
    { url = "https://files.pythonhosted.org/packages/b9/86/6e4ef77c86175bbcc2cff6e8c6a8f98a554f88ce99b9c892c9330858d07c/jiter-0.6.1-cp311-cp311-manylinux_2_17_ppc64le.manylinux2014_ppc64le.whl", hash = "sha256:e9ac7c2f092f231f5620bef23ce2e530bd218fc046098747cc390b21b8738a7a", size = 370842 },
    { url = "https://files.pythonhosted.org/packages/ba/e3/ef93fc307278d98c981b09b4f965f49312d0639ba31c2db4fe073b78a833/jiter-0.6.1-cp311-cp311-manylinux_2_17_s390x.manylinux2014_s390x.whl", hash = "sha256:e51a2d80d5fe0ffb10ed2c82b6004458be4a3f2b9c7d09ed85baa2fbf033f54b", size = 392489 },
    { url = "https://files.pythonhosted.org/packages/63/6d/bff2bce7cc17bd7e0f517490cfa4444ad94d20720eb2ccd3152a6cd57a30/jiter-0.6.1-cp311-cp311-manylinux_2_17_x86_64.manylinux2014_x86_64.whl", hash = "sha256:3343d4706a2b7140e8bd49b6c8b0a82abf9194b3f0f5925a78fc69359f8fc33c", size = 325493 },
    { url = "https://files.pythonhosted.org/packages/49/4b/56e8a5e2be5439e503b77d2c9479197e0d8199827d7f79b06592747c5210/jiter-0.6.1-cp311-cp311-manylinux_2_5_i686.manylinux1_i686.whl", hash = "sha256:82521000d18c71e41c96960cb36e915a357bc83d63a8bed63154b89d95d05ad1", size = 365974 },
    { url = "https://files.pythonhosted.org/packages/d3/9b/967752fb36ddb4b6ea7a2a8cd0ef3f167a112a2d3a2131ee544969203659/jiter-0.6.1-cp311-cp311-musllinux_1_1_aarch64.whl", hash = "sha256:3c843e7c1633470708a3987e8ce617ee2979ee18542d6eb25ae92861af3f1d62", size = 514144 },
    { url = "https://files.pythonhosted.org/packages/58/55/9b7e0021e567731b076a8bf017a1df7d6f148bb175be2ac647a0c6433bbd/jiter-0.6.1-cp311-cp311-musllinux_1_1_x86_64.whl", hash = "sha256:a2e861658c3fe849efc39b06ebb98d042e4a4c51a8d7d1c3ddc3b1ea091d0784", size = 496072 },
    { url = "https://files.pythonhosted.org/packages/ca/37/9e0638d2a129a1b72344a90a03b2b518c048066db0858aaf0877cb9d4acd/jiter-0.6.1-cp311-none-win32.whl", hash = "sha256:7d72fc86474862c9c6d1f87b921b70c362f2b7e8b2e3c798bb7d58e419a6bc0f", size = 197571 },
    { url = "https://files.pythonhosted.org/packages/65/8a/78d337464e2b2e552d2988148e3e51da5445d910345c0d00f1982fd9aad4/jiter-0.6.1-cp311-none-win_amd64.whl", hash = "sha256:3e36a320634f33a07794bb15b8da995dccb94f944d298c8cfe2bd99b1b8a574a", size = 201994 },
    { url = "https://files.pythonhosted.org/packages/2e/d5/fcdfbcea637f8b9b833597797d6b77fd7e22649b4794fc571674477c8520/jiter-0.6.1-cp312-cp312-macosx_10_12_x86_64.whl", hash = "sha256:1fad93654d5a7dcce0809aff66e883c98e2618b86656aeb2129db2cd6f26f867", size = 289279 },
    { url = "https://files.pythonhosted.org/packages/9a/47/8e4a7704a267b8d1d3287b4353fc07f1f4a3541b27988ea3e49ccbf3164a/jiter-0.6.1-cp312-cp312-macosx_11_0_arm64.whl", hash = "sha256:4e6e340e8cd92edab7f6a3a904dbbc8137e7f4b347c49a27da9814015cc0420c", size = 300931 },
    { url = "https://files.pythonhosted.org/packages/ea/4f/fbb1e11fcc3881d108359d3db8456715c9d30ddfce84dc5f9e0856e08e11/jiter-0.6.1-cp312-cp312-manylinux_2_17_aarch64.manylinux2014_aarch64.whl", hash = "sha256:691352e5653af84ed71763c3c427cff05e4d658c508172e01e9c956dfe004aba", size = 336534 },
    { url = "https://files.pythonhosted.org/packages/29/8a/4c1e1229f89127187df166de760438b2a20e5a311391ba10d2b69db0da6f/jiter-0.6.1-cp312-cp312-manylinux_2_17_armv7l.manylinux2014_armv7l.whl", hash = "sha256:defee3949313c1f5b55e18be45089970cdb936eb2a0063f5020c4185db1b63c9", size = 354266 },
    { url = "https://files.pythonhosted.org/packages/19/15/3f27f4b9d40bc7709a30fda99876cbe9e9f75a0ea2ef7d55f3dd4d04f927/jiter-0.6.1-cp312-cp312-manylinux_2_17_ppc64le.manylinux2014_ppc64le.whl", hash = "sha256:26d2bdd5da097e624081c6b5d416d3ee73e5b13f1703bcdadbb1881f0caa1933", size = 370492 },
    { url = "https://files.pythonhosted.org/packages/1f/9d/9ec03c07325bc3a3c5b5082840b8ecb7e7ad38f3071c149b7c6fb9e78706/jiter-0.6.1-cp312-cp312-manylinux_2_17_s390x.manylinux2014_s390x.whl", hash = "sha256:18aa9d1626b61c0734b973ed7088f8a3d690d0b7f5384a5270cd04f4d9f26c86", size = 390330 },
    { url = "https://files.pythonhosted.org/packages/bd/3b/612ea6daa52d64bc0cc46f2bd2e138952c58f1edbe86b17fd89e07c33d86/jiter-0.6.1-cp312-cp312-manylinux_2_17_x86_64.manylinux2014_x86_64.whl", hash = "sha256:7a3567c8228afa5ddcce950631c6b17397ed178003dc9ee7e567c4c4dcae9fa0", size = 324245 },
    { url = "https://files.pythonhosted.org/packages/21/0f/f3a1ffd9f203d4014b4e5045c0ea2c67ee71a7eee8bf3408dbf11007cf07/jiter-0.6.1-cp312-cp312-manylinux_2_5_i686.manylinux1_i686.whl", hash = "sha256:e5c0507131c922defe3f04c527d6838932fcdfd69facebafd7d3574fa3395314", size = 368232 },
    { url = "https://files.pythonhosted.org/packages/62/12/5d75729e0a57804852de0affc6f03b3df8518259e47ed4cd89aeeb671a71/jiter-0.6.1-cp312-cp312-musllinux_1_1_aarch64.whl", hash = "sha256:540fcb224d7dc1bcf82f90f2ffb652df96f2851c031adca3c8741cb91877143b", size = 513820 },
    { url = "https://files.pythonhosted.org/packages/5f/e8/e47734280e19cd465832e610e1c69367ee72947de738785c4b6fc4031e25/jiter-0.6.1-cp312-cp312-musllinux_1_1_x86_64.whl", hash = "sha256:e7b75436d4fa2032b2530ad989e4cb0ca74c655975e3ff49f91a1a3d7f4e1df2", size = 496023 },
    { url = "https://files.pythonhosted.org/packages/52/01/5f65dd1387d39aa3fd4a98a5be1d8470e929a0cb0dd6cbfebaccd9a20ac5/jiter-0.6.1-cp312-none-win32.whl", hash = "sha256:883d2ced7c21bf06874fdeecab15014c1c6d82216765ca6deef08e335fa719e0", size = 197425 },
    { url = "https://files.pythonhosted.org/packages/43/b2/bd6665030f7d7cd5d9182c62a869c3d5ceadd7bff9f1b305de9192e7dbf8/jiter-0.6.1-cp312-none-win_amd64.whl", hash = "sha256:91e63273563401aadc6c52cca64a7921c50b29372441adc104127b910e98a5b6", size = 198966 },
    { url = "https://files.pythonhosted.org/packages/23/38/7b48e0149778ff4b893567c9fd997ecfcc013e290375aa7823e1f681b3d3/jiter-0.6.1-cp313-cp313-macosx_10_12_x86_64.whl", hash = "sha256:852508a54fe3228432e56019da8b69208ea622a3069458252f725d634e955b31", size = 288674 },
    { url = "https://files.pythonhosted.org/packages/85/3b/96d15b483d82a637279da53a1d299dd5da6e029b9905bcd1a4e1f89b8e4f/jiter-0.6.1-cp313-cp313-macosx_11_0_arm64.whl", hash = "sha256:f491cc69ff44e5a1e8bc6bf2b94c1f98d179e1aaf4a554493c171a5b2316b701", size = 301531 },
    { url = "https://files.pythonhosted.org/packages/cf/54/9681f112cbec4e197259e9db679bd4bc314f4bd24f74b9aa5e93073990b5/jiter-0.6.1-cp313-cp313-manylinux_2_17_aarch64.manylinux2014_aarch64.whl", hash = "sha256:cc56c8f0b2a28ad4d8047f3ae62d25d0e9ae01b99940ec0283263a04724de1f3", size = 335954 },
    { url = "https://files.pythonhosted.org/packages/4a/4d/f9c0ba82b154c66278e28348086086264ccf50622ae468ec215e4bbc2873/jiter-0.6.1-cp313-cp313-manylinux_2_17_armv7l.manylinux2014_armv7l.whl", hash = "sha256:51b58f7a0d9e084a43b28b23da2b09fc5e8df6aa2b6a27de43f991293cab85fd", size = 353996 },
    { url = "https://files.pythonhosted.org/packages/ee/be/7f26b258ef190f6d582e21c76c7dd1097753a2203bad3e1643f45392720a/jiter-0.6.1-cp313-cp313-manylinux_2_17_ppc64le.manylinux2014_ppc64le.whl", hash = "sha256:5f79ce15099154c90ef900d69c6b4c686b64dfe23b0114e0971f2fecd306ec6c", size = 369733 },
    { url = "https://files.pythonhosted.org/packages/5f/85/037ed5261fa622312471ef5520b2135c26b29256c83adc16c8cc55dc4108/jiter-0.6.1-cp313-cp313-manylinux_2_17_s390x.manylinux2014_s390x.whl", hash = "sha256:03a025b52009f47e53ea619175d17e4ded7c035c6fbd44935cb3ada11e1fd592", size = 389920 },
    { url = "https://files.pythonhosted.org/packages/a8/f3/2e01294712faa476be9e6ceb49e424c3919e03415ded76d103378a06bb80/jiter-0.6.1-cp313-cp313-manylinux_2_17_x86_64.manylinux2014_x86_64.whl", hash = "sha256:c74a8d93718137c021d9295248a87c2f9fdc0dcafead12d2930bc459ad40f885", size = 324138 },
    { url = "https://files.pythonhosted.org/packages/00/45/50377814f21b6412c7785be27f2dace225af52e0af20be7af899a7e3f264/jiter-0.6.1-cp313-cp313-manylinux_2_5_i686.manylinux1_i686.whl", hash = "sha256:40b03b75f903975f68199fc4ec73d546150919cb7e534f3b51e727c4d6ccca5a", size = 367610 },
    { url = "https://files.pythonhosted.org/packages/af/fc/51ba30875125381bfe21a1572c176de1a7dd64a386a7498355fc100decc4/jiter-0.6.1-cp313-cp313-musllinux_1_1_aarch64.whl", hash = "sha256:825651a3f04cf92a661d22cad61fc913400e33aa89b3e3ad9a6aa9dc8a1f5a71", size = 512945 },
    { url = "https://files.pythonhosted.org/packages/69/60/af26168bd4916f9199ed433161e9f8a4eeda581a4e5982560d0f22dd146c/jiter-0.6.1-cp313-cp313-musllinux_1_1_x86_64.whl", hash = "sha256:928bf25eb69ddb292ab8177fe69d3fbf76c7feab5fce1c09265a7dccf25d3991", size = 494963 },
    { url = "https://files.pythonhosted.org/packages/f3/2f/4f3cc5c9067a6fd1020d3c4365546535a69ed77da7fba2bec24368f3662c/jiter-0.6.1-cp313-none-win32.whl", hash = "sha256:352cd24121e80d3d053fab1cc9806258cad27c53cad99b7a3cac57cf934b12e4", size = 196869 },
    { url = "https://files.pythonhosted.org/packages/7a/fc/8709ee90837e94790d8b50db51c7b8a70e86e41b2c81e824c20b0ecfeba7/jiter-0.6.1-cp313-none-win_amd64.whl", hash = "sha256:be7503dd6f4bf02c2a9bacb5cc9335bc59132e7eee9d3e931b13d76fd80d7fda", size = 198919 },
]

[[package]]
name = "m3u8"
version = "6.0.0"
source = { registry = "https://pypi.org/simple" }
sdist = { url = "https://files.pythonhosted.org/packages/9b/a5/73697aaa99bb32b610adc1f11d46a0c0c370351292e9b271755084a145e6/m3u8-6.0.0.tar.gz", hash = "sha256:7ade990a1667d7a653bcaf9413b16c3eb5cd618982ff46aaff57fe6d9fa9c0fd", size = 42720 }
wheels = [
    { url = "https://files.pythonhosted.org/packages/f8/31/50f3c38b38ff28635ff9c4a4afefddccc5f1b57457b539bdbdf75ce18669/m3u8-6.0.0-py3-none-any.whl", hash = "sha256:566d0748739c552dad10f8c87150078de6a0ec25071fa48e6968e96fc6dcba5d", size = 24133 },
]

[[package]]
name = "markdown-it-py"
version = "3.0.0"
source = { registry = "https://pypi.org/simple" }
dependencies = [
    { name = "mdurl" },
]
sdist = { url = "https://files.pythonhosted.org/packages/38/71/3b932df36c1a044d397a1f92d1cf91ee0a503d91e470cbd670aa66b07ed0/markdown-it-py-3.0.0.tar.gz", hash = "sha256:e3f60a94fa066dc52ec76661e37c851cb232d92f9886b15cb560aaada2df8feb", size = 74596 }
wheels = [
    { url = "https://files.pythonhosted.org/packages/42/d7/1ec15b46af6af88f19b8e5ffea08fa375d433c998b8a7639e76935c14f1f/markdown_it_py-3.0.0-py3-none-any.whl", hash = "sha256:355216845c60bd96232cd8d8c40e8f9765cc86f46880e43a8fd22dc1a1a8cab1", size = 87528 },
]

[[package]]
name = "markupsafe"
version = "3.0.2"
source = { registry = "https://pypi.org/simple" }
sdist = { url = "https://files.pythonhosted.org/packages/b2/97/5d42485e71dfc078108a86d6de8fa46db44a1a9295e89c5d6d4a06e23a62/markupsafe-3.0.2.tar.gz", hash = "sha256:ee55d3edf80167e48ea11a923c7386f4669df67d7994554387f84e7d8b0a2bf0", size = 20537 }
wheels = [
    { url = "https://files.pythonhosted.org/packages/6b/28/bbf83e3f76936960b850435576dd5e67034e200469571be53f69174a2dfd/MarkupSafe-3.0.2-cp311-cp311-macosx_10_9_universal2.whl", hash = "sha256:9025b4018f3a1314059769c7bf15441064b2207cb3f065e6ea1e7359cb46db9d", size = 14353 },
    { url = "https://files.pythonhosted.org/packages/6c/30/316d194b093cde57d448a4c3209f22e3046c5bb2fb0820b118292b334be7/MarkupSafe-3.0.2-cp311-cp311-macosx_11_0_arm64.whl", hash = "sha256:93335ca3812df2f366e80509ae119189886b0f3c2b81325d39efdb84a1e2ae93", size = 12392 },
    { url = "https://files.pythonhosted.org/packages/f2/96/9cdafba8445d3a53cae530aaf83c38ec64c4d5427d975c974084af5bc5d2/MarkupSafe-3.0.2-cp311-cp311-manylinux_2_17_aarch64.manylinux2014_aarch64.whl", hash = "sha256:2cb8438c3cbb25e220c2ab33bb226559e7afb3baec11c4f218ffa7308603c832", size = 23984 },
    { url = "https://files.pythonhosted.org/packages/f1/a4/aefb044a2cd8d7334c8a47d3fb2c9f328ac48cb349468cc31c20b539305f/MarkupSafe-3.0.2-cp311-cp311-manylinux_2_17_x86_64.manylinux2014_x86_64.whl", hash = "sha256:a123e330ef0853c6e822384873bef7507557d8e4a082961e1defa947aa59ba84", size = 23120 },
    { url = "https://files.pythonhosted.org/packages/8d/21/5e4851379f88f3fad1de30361db501300d4f07bcad047d3cb0449fc51f8c/MarkupSafe-3.0.2-cp311-cp311-manylinux_2_5_i686.manylinux1_i686.manylinux_2_17_i686.manylinux2014_i686.whl", hash = "sha256:1e084f686b92e5b83186b07e8a17fc09e38fff551f3602b249881fec658d3eca", size = 23032 },
    { url = "https://files.pythonhosted.org/packages/00/7b/e92c64e079b2d0d7ddf69899c98842f3f9a60a1ae72657c89ce2655c999d/MarkupSafe-3.0.2-cp311-cp311-musllinux_1_2_aarch64.whl", hash = "sha256:d8213e09c917a951de9d09ecee036d5c7d36cb6cb7dbaece4c71a60d79fb9798", size = 24057 },
    { url = "https://files.pythonhosted.org/packages/f9/ac/46f960ca323037caa0a10662ef97d0a4728e890334fc156b9f9e52bcc4ca/MarkupSafe-3.0.2-cp311-cp311-musllinux_1_2_i686.whl", hash = "sha256:5b02fb34468b6aaa40dfc198d813a641e3a63b98c2b05a16b9f80b7ec314185e", size = 23359 },
    { url = "https://files.pythonhosted.org/packages/69/84/83439e16197337b8b14b6a5b9c2105fff81d42c2a7c5b58ac7b62ee2c3b1/MarkupSafe-3.0.2-cp311-cp311-musllinux_1_2_x86_64.whl", hash = "sha256:0bff5e0ae4ef2e1ae4fdf2dfd5b76c75e5c2fa4132d05fc1b0dabcd20c7e28c4", size = 23306 },
    { url = "https://files.pythonhosted.org/packages/9a/34/a15aa69f01e2181ed8d2b685c0d2f6655d5cca2c4db0ddea775e631918cd/MarkupSafe-3.0.2-cp311-cp311-win32.whl", hash = "sha256:6c89876f41da747c8d3677a2b540fb32ef5715f97b66eeb0c6b66f5e3ef6f59d", size = 15094 },
    { url = "https://files.pythonhosted.org/packages/da/b8/3a3bd761922d416f3dc5d00bfbed11f66b1ab89a0c2b6e887240a30b0f6b/MarkupSafe-3.0.2-cp311-cp311-win_amd64.whl", hash = "sha256:70a87b411535ccad5ef2f1df5136506a10775d267e197e4cf531ced10537bd6b", size = 15521 },
    { url = "https://files.pythonhosted.org/packages/22/09/d1f21434c97fc42f09d290cbb6350d44eb12f09cc62c9476effdb33a18aa/MarkupSafe-3.0.2-cp312-cp312-macosx_10_13_universal2.whl", hash = "sha256:9778bd8ab0a994ebf6f84c2b949e65736d5575320a17ae8984a77fab08db94cf", size = 14274 },
    { url = "https://files.pythonhosted.org/packages/6b/b0/18f76bba336fa5aecf79d45dcd6c806c280ec44538b3c13671d49099fdd0/MarkupSafe-3.0.2-cp312-cp312-macosx_11_0_arm64.whl", hash = "sha256:846ade7b71e3536c4e56b386c2a47adf5741d2d8b94ec9dc3e92e5e1ee1e2225", size = 12348 },
    { url = "https://files.pythonhosted.org/packages/e0/25/dd5c0f6ac1311e9b40f4af06c78efde0f3b5cbf02502f8ef9501294c425b/MarkupSafe-3.0.2-cp312-cp312-manylinux_2_17_aarch64.manylinux2014_aarch64.whl", hash = "sha256:1c99d261bd2d5f6b59325c92c73df481e05e57f19837bdca8413b9eac4bd8028", size = 24149 },
    { url = "https://files.pythonhosted.org/packages/f3/f0/89e7aadfb3749d0f52234a0c8c7867877876e0a20b60e2188e9850794c17/MarkupSafe-3.0.2-cp312-cp312-manylinux_2_17_x86_64.manylinux2014_x86_64.whl", hash = "sha256:e17c96c14e19278594aa4841ec148115f9c7615a47382ecb6b82bd8fea3ab0c8", size = 23118 },
    { url = "https://files.pythonhosted.org/packages/d5/da/f2eeb64c723f5e3777bc081da884b414671982008c47dcc1873d81f625b6/MarkupSafe-3.0.2-cp312-cp312-manylinux_2_5_i686.manylinux1_i686.manylinux_2_17_i686.manylinux2014_i686.whl", hash = "sha256:88416bd1e65dcea10bc7569faacb2c20ce071dd1f87539ca2ab364bf6231393c", size = 22993 },
    { url = "https://files.pythonhosted.org/packages/da/0e/1f32af846df486dce7c227fe0f2398dc7e2e51d4a370508281f3c1c5cddc/MarkupSafe-3.0.2-cp312-cp312-musllinux_1_2_aarch64.whl", hash = "sha256:2181e67807fc2fa785d0592dc2d6206c019b9502410671cc905d132a92866557", size = 24178 },
    { url = "https://files.pythonhosted.org/packages/c4/f6/bb3ca0532de8086cbff5f06d137064c8410d10779c4c127e0e47d17c0b71/MarkupSafe-3.0.2-cp312-cp312-musllinux_1_2_i686.whl", hash = "sha256:52305740fe773d09cffb16f8ed0427942901f00adedac82ec8b67752f58a1b22", size = 23319 },
    { url = "https://files.pythonhosted.org/packages/a2/82/8be4c96ffee03c5b4a034e60a31294daf481e12c7c43ab8e34a1453ee48b/MarkupSafe-3.0.2-cp312-cp312-musllinux_1_2_x86_64.whl", hash = "sha256:ad10d3ded218f1039f11a75f8091880239651b52e9bb592ca27de44eed242a48", size = 23352 },
    { url = "https://files.pythonhosted.org/packages/51/ae/97827349d3fcffee7e184bdf7f41cd6b88d9919c80f0263ba7acd1bbcb18/MarkupSafe-3.0.2-cp312-cp312-win32.whl", hash = "sha256:0f4ca02bea9a23221c0182836703cbf8930c5e9454bacce27e767509fa286a30", size = 15097 },
    { url = "https://files.pythonhosted.org/packages/c1/80/a61f99dc3a936413c3ee4e1eecac96c0da5ed07ad56fd975f1a9da5bc630/MarkupSafe-3.0.2-cp312-cp312-win_amd64.whl", hash = "sha256:8e06879fc22a25ca47312fbe7c8264eb0b662f6db27cb2d3bbbc74b1df4b9b87", size = 15601 },
    { url = "https://files.pythonhosted.org/packages/83/0e/67eb10a7ecc77a0c2bbe2b0235765b98d164d81600746914bebada795e97/MarkupSafe-3.0.2-cp313-cp313-macosx_10_13_universal2.whl", hash = "sha256:ba9527cdd4c926ed0760bc301f6728ef34d841f405abf9d4f959c478421e4efd", size = 14274 },
    { url = "https://files.pythonhosted.org/packages/2b/6d/9409f3684d3335375d04e5f05744dfe7e9f120062c9857df4ab490a1031a/MarkupSafe-3.0.2-cp313-cp313-macosx_11_0_arm64.whl", hash = "sha256:f8b3d067f2e40fe93e1ccdd6b2e1d16c43140e76f02fb1319a05cf2b79d99430", size = 12352 },
    { url = "https://files.pythonhosted.org/packages/d2/f5/6eadfcd3885ea85fe2a7c128315cc1bb7241e1987443d78c8fe712d03091/MarkupSafe-3.0.2-cp313-cp313-manylinux_2_17_aarch64.manylinux2014_aarch64.whl", hash = "sha256:569511d3b58c8791ab4c2e1285575265991e6d8f8700c7be0e88f86cb0672094", size = 24122 },
    { url = "https://files.pythonhosted.org/packages/0c/91/96cf928db8236f1bfab6ce15ad070dfdd02ed88261c2afafd4b43575e9e9/MarkupSafe-3.0.2-cp313-cp313-manylinux_2_17_x86_64.manylinux2014_x86_64.whl", hash = "sha256:15ab75ef81add55874e7ab7055e9c397312385bd9ced94920f2802310c930396", size = 23085 },
    { url = "https://files.pythonhosted.org/packages/c2/cf/c9d56af24d56ea04daae7ac0940232d31d5a8354f2b457c6d856b2057d69/MarkupSafe-3.0.2-cp313-cp313-manylinux_2_5_i686.manylinux1_i686.manylinux_2_17_i686.manylinux2014_i686.whl", hash = "sha256:f3818cb119498c0678015754eba762e0d61e5b52d34c8b13d770f0719f7b1d79", size = 22978 },
    { url = "https://files.pythonhosted.org/packages/2a/9f/8619835cd6a711d6272d62abb78c033bda638fdc54c4e7f4272cf1c0962b/MarkupSafe-3.0.2-cp313-cp313-musllinux_1_2_aarch64.whl", hash = "sha256:cdb82a876c47801bb54a690c5ae105a46b392ac6099881cdfb9f6e95e4014c6a", size = 24208 },
    { url = "https://files.pythonhosted.org/packages/f9/bf/176950a1792b2cd2102b8ffeb5133e1ed984547b75db47c25a67d3359f77/MarkupSafe-3.0.2-cp313-cp313-musllinux_1_2_i686.whl", hash = "sha256:cabc348d87e913db6ab4aa100f01b08f481097838bdddf7c7a84b7575b7309ca", size = 23357 },
    { url = "https://files.pythonhosted.org/packages/ce/4f/9a02c1d335caabe5c4efb90e1b6e8ee944aa245c1aaaab8e8a618987d816/MarkupSafe-3.0.2-cp313-cp313-musllinux_1_2_x86_64.whl", hash = "sha256:444dcda765c8a838eaae23112db52f1efaf750daddb2d9ca300bcae1039adc5c", size = 23344 },
    { url = "https://files.pythonhosted.org/packages/ee/55/c271b57db36f748f0e04a759ace9f8f759ccf22b4960c270c78a394f58be/MarkupSafe-3.0.2-cp313-cp313-win32.whl", hash = "sha256:bcf3e58998965654fdaff38e58584d8937aa3096ab5354d493c77d1fdd66d7a1", size = 15101 },
    { url = "https://files.pythonhosted.org/packages/29/88/07df22d2dd4df40aba9f3e402e6dc1b8ee86297dddbad4872bd5e7b0094f/MarkupSafe-3.0.2-cp313-cp313-win_amd64.whl", hash = "sha256:e6a2a455bd412959b57a172ce6328d2dd1f01cb2135efda2e4576e8a23fa3b0f", size = 15603 },
    { url = "https://files.pythonhosted.org/packages/62/6a/8b89d24db2d32d433dffcd6a8779159da109842434f1dd2f6e71f32f738c/MarkupSafe-3.0.2-cp313-cp313t-macosx_10_13_universal2.whl", hash = "sha256:b5a6b3ada725cea8a5e634536b1b01c30bcdcd7f9c6fff4151548d5bf6b3a36c", size = 14510 },
    { url = "https://files.pythonhosted.org/packages/7a/06/a10f955f70a2e5a9bf78d11a161029d278eeacbd35ef806c3fd17b13060d/MarkupSafe-3.0.2-cp313-cp313t-macosx_11_0_arm64.whl", hash = "sha256:a904af0a6162c73e3edcb969eeeb53a63ceeb5d8cf642fade7d39e7963a22ddb", size = 12486 },
    { url = "https://files.pythonhosted.org/packages/34/cf/65d4a571869a1a9078198ca28f39fba5fbb910f952f9dbc5220afff9f5e6/MarkupSafe-3.0.2-cp313-cp313t-manylinux_2_17_aarch64.manylinux2014_aarch64.whl", hash = "sha256:4aa4e5faecf353ed117801a068ebab7b7e09ffb6e1d5e412dc852e0da018126c", size = 25480 },
    { url = "https://files.pythonhosted.org/packages/0c/e3/90e9651924c430b885468b56b3d597cabf6d72be4b24a0acd1fa0e12af67/MarkupSafe-3.0.2-cp313-cp313t-manylinux_2_17_x86_64.manylinux2014_x86_64.whl", hash = "sha256:c0ef13eaeee5b615fb07c9a7dadb38eac06a0608b41570d8ade51c56539e509d", size = 23914 },
    { url = "https://files.pythonhosted.org/packages/66/8c/6c7cf61f95d63bb866db39085150df1f2a5bd3335298f14a66b48e92659c/MarkupSafe-3.0.2-cp313-cp313t-manylinux_2_5_i686.manylinux1_i686.manylinux_2_17_i686.manylinux2014_i686.whl", hash = "sha256:d16a81a06776313e817c951135cf7340a3e91e8c1ff2fac444cfd75fffa04afe", size = 23796 },
    { url = "https://files.pythonhosted.org/packages/bb/35/cbe9238ec3f47ac9a7c8b3df7a808e7cb50fe149dc7039f5f454b3fba218/MarkupSafe-3.0.2-cp313-cp313t-musllinux_1_2_aarch64.whl", hash = "sha256:6381026f158fdb7c72a168278597a5e3a5222e83ea18f543112b2662a9b699c5", size = 25473 },
    { url = "https://files.pythonhosted.org/packages/e6/32/7621a4382488aa283cc05e8984a9c219abad3bca087be9ec77e89939ded9/MarkupSafe-3.0.2-cp313-cp313t-musllinux_1_2_i686.whl", hash = "sha256:3d79d162e7be8f996986c064d1c7c817f6df3a77fe3d6859f6f9e7be4b8c213a", size = 24114 },
    { url = "https://files.pythonhosted.org/packages/0d/80/0985960e4b89922cb5a0bac0ed39c5b96cbc1a536a99f30e8c220a996ed9/MarkupSafe-3.0.2-cp313-cp313t-musllinux_1_2_x86_64.whl", hash = "sha256:131a3c7689c85f5ad20f9f6fb1b866f402c445b220c19fe4308c0b147ccd2ad9", size = 24098 },
    { url = "https://files.pythonhosted.org/packages/82/78/fedb03c7d5380df2427038ec8d973587e90561b2d90cd472ce9254cf348b/MarkupSafe-3.0.2-cp313-cp313t-win32.whl", hash = "sha256:ba8062ed2cf21c07a9e295d5b8a2a5ce678b913b45fdf68c32d95d6c1291e0b6", size = 15208 },
    { url = "https://files.pythonhosted.org/packages/4f/65/6079a46068dfceaeabb5dcad6d674f5f5c61a6fa5673746f42a9f4c233b3/MarkupSafe-3.0.2-cp313-cp313t-win_amd64.whl", hash = "sha256:e444a31f8db13eb18ada366ab3cf45fd4b31e4db1236a4448f68778c1d1a5a2f", size = 15739 },
]

[[package]]
name = "mdurl"
version = "0.1.2"
source = { registry = "https://pypi.org/simple" }
sdist = { url = "https://files.pythonhosted.org/packages/d6/54/cfe61301667036ec958cb99bd3efefba235e65cdeb9c84d24a8293ba1d90/mdurl-0.1.2.tar.gz", hash = "sha256:bb413d29f5eea38f31dd4754dd7377d4465116fb207585f97bf925588687c1ba", size = 8729 }
wheels = [
    { url = "https://files.pythonhosted.org/packages/b3/38/89ba8ad64ae25be8de66a6d463314cf1eb366222074cfda9ee839c56a4b4/mdurl-0.1.2-py3-none-any.whl", hash = "sha256:84008a41e51615a49fc9966191ff91509e3c40b939176e643fd50a5c2196b8f8", size = 9979 },
]

[[package]]
name = "multidict"
version = "6.1.0"
source = { registry = "https://pypi.org/simple" }
sdist = { url = "https://files.pythonhosted.org/packages/d6/be/504b89a5e9ca731cd47487e91c469064f8ae5af93b7259758dcfc2b9c848/multidict-6.1.0.tar.gz", hash = "sha256:22ae2ebf9b0c69d206c003e2f6a914ea33f0a932d4aa16f236afc049d9958f4a", size = 64002 }
wheels = [
    { url = "https://files.pythonhosted.org/packages/93/13/df3505a46d0cd08428e4c8169a196131d1b0c4b515c3649829258843dde6/multidict-6.1.0-cp311-cp311-macosx_10_9_universal2.whl", hash = "sha256:3efe2c2cb5763f2f1b275ad2bf7a287d3f7ebbef35648a9726e3b69284a4f3d6", size = 48570 },
    { url = "https://files.pythonhosted.org/packages/f0/e1/a215908bfae1343cdb72f805366592bdd60487b4232d039c437fe8f5013d/multidict-6.1.0-cp311-cp311-macosx_10_9_x86_64.whl", hash = "sha256:c7053d3b0353a8b9de430a4f4b4268ac9a4fb3481af37dfe49825bf45ca24156", size = 29316 },
    { url = "https://files.pythonhosted.org/packages/70/0f/6dc70ddf5d442702ed74f298d69977f904960b82368532c88e854b79f72b/multidict-6.1.0-cp311-cp311-macosx_11_0_arm64.whl", hash = "sha256:27e5fc84ccef8dfaabb09d82b7d179c7cf1a3fbc8a966f8274fcb4ab2eb4cadb", size = 29640 },
    { url = "https://files.pythonhosted.org/packages/d8/6d/9c87b73a13d1cdea30b321ef4b3824449866bd7f7127eceed066ccb9b9ff/multidict-6.1.0-cp311-cp311-manylinux_2_17_aarch64.manylinux2014_aarch64.whl", hash = "sha256:0e2b90b43e696f25c62656389d32236e049568b39320e2735d51f08fd362761b", size = 131067 },
    { url = "https://files.pythonhosted.org/packages/cc/1e/1b34154fef373371fd6c65125b3d42ff5f56c7ccc6bfff91b9b3c60ae9e0/multidict-6.1.0-cp311-cp311-manylinux_2_17_ppc64le.manylinux2014_ppc64le.whl", hash = "sha256:d83a047959d38a7ff552ff94be767b7fd79b831ad1cd9920662db05fec24fe72", size = 138507 },
    { url = "https://files.pythonhosted.org/packages/fb/e0/0bc6b2bac6e461822b5f575eae85da6aae76d0e2a79b6665d6206b8e2e48/multidict-6.1.0-cp311-cp311-manylinux_2_17_s390x.manylinux2014_s390x.whl", hash = "sha256:d1a9dd711d0877a1ece3d2e4fea11a8e75741ca21954c919406b44e7cf971304", size = 133905 },
    { url = "https://files.pythonhosted.org/packages/ba/af/73d13b918071ff9b2205fcf773d316e0f8fefb4ec65354bbcf0b10908cc6/multidict-6.1.0-cp311-cp311-manylinux_2_17_x86_64.manylinux2014_x86_64.whl", hash = "sha256:ec2abea24d98246b94913b76a125e855eb5c434f7c46546046372fe60f666351", size = 129004 },
    { url = "https://files.pythonhosted.org/packages/74/21/23960627b00ed39643302d81bcda44c9444ebcdc04ee5bedd0757513f259/multidict-6.1.0-cp311-cp311-manylinux_2_5_i686.manylinux1_i686.manylinux_2_17_i686.manylinux2014_i686.whl", hash = "sha256:4867cafcbc6585e4b678876c489b9273b13e9fff9f6d6d66add5e15d11d926cb", size = 121308 },
    { url = "https://files.pythonhosted.org/packages/8b/5c/cf282263ffce4a596ed0bb2aa1a1dddfe1996d6a62d08842a8d4b33dca13/multidict-6.1.0-cp311-cp311-musllinux_1_2_aarch64.whl", hash = "sha256:5b48204e8d955c47c55b72779802b219a39acc3ee3d0116d5080c388970b76e3", size = 132608 },
    { url = "https://files.pythonhosted.org/packages/d7/3e/97e778c041c72063f42b290888daff008d3ab1427f5b09b714f5a8eff294/multidict-6.1.0-cp311-cp311-musllinux_1_2_i686.whl", hash = "sha256:d8fff389528cad1618fb4b26b95550327495462cd745d879a8c7c2115248e399", size = 127029 },
    { url = "https://files.pythonhosted.org/packages/47/ac/3efb7bfe2f3aefcf8d103e9a7162572f01936155ab2f7ebcc7c255a23212/multidict-6.1.0-cp311-cp311-musllinux_1_2_ppc64le.whl", hash = "sha256:a7a9541cd308eed5e30318430a9c74d2132e9a8cb46b901326272d780bf2d423", size = 137594 },
    { url = "https://files.pythonhosted.org/packages/42/9b/6c6e9e8dc4f915fc90a9b7798c44a30773dea2995fdcb619870e705afe2b/multidict-6.1.0-cp311-cp311-musllinux_1_2_s390x.whl", hash = "sha256:da1758c76f50c39a2efd5e9859ce7d776317eb1dd34317c8152ac9251fc574a3", size = 134556 },
    { url = "https://files.pythonhosted.org/packages/1d/10/8e881743b26aaf718379a14ac58572a240e8293a1c9d68e1418fb11c0f90/multidict-6.1.0-cp311-cp311-musllinux_1_2_x86_64.whl", hash = "sha256:c943a53e9186688b45b323602298ab727d8865d8c9ee0b17f8d62d14b56f0753", size = 130993 },
    { url = "https://files.pythonhosted.org/packages/45/84/3eb91b4b557442802d058a7579e864b329968c8d0ea57d907e7023c677f2/multidict-6.1.0-cp311-cp311-win32.whl", hash = "sha256:90f8717cb649eea3504091e640a1b8568faad18bd4b9fcd692853a04475a4b80", size = 26405 },
    { url = "https://files.pythonhosted.org/packages/9f/0b/ad879847ecbf6d27e90a6eabb7eff6b62c129eefe617ea45eae7c1f0aead/multidict-6.1.0-cp311-cp311-win_amd64.whl", hash = "sha256:82176036e65644a6cc5bd619f65f6f19781e8ec2e5330f51aa9ada7504cc1926", size = 28795 },
    { url = "https://files.pythonhosted.org/packages/fd/16/92057c74ba3b96d5e211b553895cd6dc7cc4d1e43d9ab8fafc727681ef71/multidict-6.1.0-cp312-cp312-macosx_10_9_universal2.whl", hash = "sha256:b04772ed465fa3cc947db808fa306d79b43e896beb677a56fb2347ca1a49c1fa", size = 48713 },
    { url = "https://files.pythonhosted.org/packages/94/3d/37d1b8893ae79716179540b89fc6a0ee56b4a65fcc0d63535c6f5d96f217/multidict-6.1.0-cp312-cp312-macosx_10_9_x86_64.whl", hash = "sha256:6180c0ae073bddeb5a97a38c03f30c233e0a4d39cd86166251617d1bbd0af436", size = 29516 },
    { url = "https://files.pythonhosted.org/packages/a2/12/adb6b3200c363062f805275b4c1e656be2b3681aada66c80129932ff0bae/multidict-6.1.0-cp312-cp312-macosx_11_0_arm64.whl", hash = "sha256:071120490b47aa997cca00666923a83f02c7fbb44f71cf7f136df753f7fa8761", size = 29557 },
    { url = "https://files.pythonhosted.org/packages/47/e9/604bb05e6e5bce1e6a5cf80a474e0f072e80d8ac105f1b994a53e0b28c42/multidict-6.1.0-cp312-cp312-manylinux_2_17_aarch64.manylinux2014_aarch64.whl", hash = "sha256:50b3a2710631848991d0bf7de077502e8994c804bb805aeb2925a981de58ec2e", size = 130170 },
    { url = "https://files.pythonhosted.org/packages/7e/13/9efa50801785eccbf7086b3c83b71a4fb501a4d43549c2f2f80b8787d69f/multidict-6.1.0-cp312-cp312-manylinux_2_17_ppc64le.manylinux2014_ppc64le.whl", hash = "sha256:b58c621844d55e71c1b7f7c498ce5aa6985d743a1a59034c57a905b3f153c1ef", size = 134836 },
    { url = "https://files.pythonhosted.org/packages/bf/0f/93808b765192780d117814a6dfcc2e75de6dcc610009ad408b8814dca3ba/multidict-6.1.0-cp312-cp312-manylinux_2_17_s390x.manylinux2014_s390x.whl", hash = "sha256:55b6d90641869892caa9ca42ff913f7ff1c5ece06474fbd32fb2cf6834726c95", size = 133475 },
    { url = "https://files.pythonhosted.org/packages/d3/c8/529101d7176fe7dfe1d99604e48d69c5dfdcadb4f06561f465c8ef12b4df/multidict-6.1.0-cp312-cp312-manylinux_2_17_x86_64.manylinux2014_x86_64.whl", hash = "sha256:4b820514bfc0b98a30e3d85462084779900347e4d49267f747ff54060cc33925", size = 131049 },
    { url = "https://files.pythonhosted.org/packages/ca/0c/fc85b439014d5a58063e19c3a158a889deec399d47b5269a0f3b6a2e28bc/multidict-6.1.0-cp312-cp312-manylinux_2_5_i686.manylinux1_i686.manylinux_2_17_i686.manylinux2014_i686.whl", hash = "sha256:10a9b09aba0c5b48c53761b7c720aaaf7cf236d5fe394cd399c7ba662d5f9966", size = 120370 },
    { url = "https://files.pythonhosted.org/packages/db/46/d4416eb20176492d2258fbd47b4abe729ff3b6e9c829ea4236f93c865089/multidict-6.1.0-cp312-cp312-musllinux_1_2_aarch64.whl", hash = "sha256:1e16bf3e5fc9f44632affb159d30a437bfe286ce9e02754759be5536b169b305", size = 125178 },
    { url = "https://files.pythonhosted.org/packages/5b/46/73697ad7ec521df7de5531a32780bbfd908ded0643cbe457f981a701457c/multidict-6.1.0-cp312-cp312-musllinux_1_2_i686.whl", hash = "sha256:76f364861c3bfc98cbbcbd402d83454ed9e01a5224bb3a28bf70002a230f73e2", size = 119567 },
    { url = "https://files.pythonhosted.org/packages/cd/ed/51f060e2cb0e7635329fa6ff930aa5cffa17f4c7f5c6c3ddc3500708e2f2/multidict-6.1.0-cp312-cp312-musllinux_1_2_ppc64le.whl", hash = "sha256:820c661588bd01a0aa62a1283f20d2be4281b086f80dad9e955e690c75fb54a2", size = 129822 },
    { url = "https://files.pythonhosted.org/packages/df/9e/ee7d1954b1331da3eddea0c4e08d9142da5f14b1321c7301f5014f49d492/multidict-6.1.0-cp312-cp312-musllinux_1_2_s390x.whl", hash = "sha256:0e5f362e895bc5b9e67fe6e4ded2492d8124bdf817827f33c5b46c2fe3ffaca6", size = 128656 },
    { url = "https://files.pythonhosted.org/packages/77/00/8538f11e3356b5d95fa4b024aa566cde7a38aa7a5f08f4912b32a037c5dc/multidict-6.1.0-cp312-cp312-musllinux_1_2_x86_64.whl", hash = "sha256:3ec660d19bbc671e3a6443325f07263be452c453ac9e512f5eb935e7d4ac28b3", size = 125360 },
    { url = "https://files.pythonhosted.org/packages/be/05/5d334c1f2462d43fec2363cd00b1c44c93a78c3925d952e9a71caf662e96/multidict-6.1.0-cp312-cp312-win32.whl", hash = "sha256:58130ecf8f7b8112cdb841486404f1282b9c86ccb30d3519faf301b2e5659133", size = 26382 },
    { url = "https://files.pythonhosted.org/packages/a3/bf/f332a13486b1ed0496d624bcc7e8357bb8053823e8cd4b9a18edc1d97e73/multidict-6.1.0-cp312-cp312-win_amd64.whl", hash = "sha256:188215fc0aafb8e03341995e7c4797860181562380f81ed0a87ff455b70bf1f1", size = 28529 },
    { url = "https://files.pythonhosted.org/packages/22/67/1c7c0f39fe069aa4e5d794f323be24bf4d33d62d2a348acdb7991f8f30db/multidict-6.1.0-cp313-cp313-macosx_10_13_universal2.whl", hash = "sha256:d569388c381b24671589335a3be6e1d45546c2988c2ebe30fdcada8457a31008", size = 48771 },
    { url = "https://files.pythonhosted.org/packages/3c/25/c186ee7b212bdf0df2519eacfb1981a017bda34392c67542c274651daf23/multidict-6.1.0-cp313-cp313-macosx_10_13_x86_64.whl", hash = "sha256:052e10d2d37810b99cc170b785945421141bf7bb7d2f8799d431e7db229c385f", size = 29533 },
    { url = "https://files.pythonhosted.org/packages/67/5e/04575fd837e0958e324ca035b339cea174554f6f641d3fb2b4f2e7ff44a2/multidict-6.1.0-cp313-cp313-macosx_11_0_arm64.whl", hash = "sha256:f90c822a402cb865e396a504f9fc8173ef34212a342d92e362ca498cad308e28", size = 29595 },
    { url = "https://files.pythonhosted.org/packages/d3/b2/e56388f86663810c07cfe4a3c3d87227f3811eeb2d08450b9e5d19d78876/multidict-6.1.0-cp313-cp313-manylinux_2_17_aarch64.manylinux2014_aarch64.whl", hash = "sha256:b225d95519a5bf73860323e633a664b0d85ad3d5bede6d30d95b35d4dfe8805b", size = 130094 },
    { url = "https://files.pythonhosted.org/packages/6c/ee/30ae9b4186a644d284543d55d491fbd4239b015d36b23fea43b4c94f7052/multidict-6.1.0-cp313-cp313-manylinux_2_17_ppc64le.manylinux2014_ppc64le.whl", hash = "sha256:23bfd518810af7de1116313ebd9092cb9aa629beb12f6ed631ad53356ed6b86c", size = 134876 },
    { url = "https://files.pythonhosted.org/packages/84/c7/70461c13ba8ce3c779503c70ec9d0345ae84de04521c1f45a04d5f48943d/multidict-6.1.0-cp313-cp313-manylinux_2_17_s390x.manylinux2014_s390x.whl", hash = "sha256:5c09fcfdccdd0b57867577b719c69e347a436b86cd83747f179dbf0cc0d4c1f3", size = 133500 },
    { url = "https://files.pythonhosted.org/packages/4a/9f/002af221253f10f99959561123fae676148dd730e2daa2cd053846a58507/multidict-6.1.0-cp313-cp313-manylinux_2_17_x86_64.manylinux2014_x86_64.whl", hash = "sha256:bf6bea52ec97e95560af5ae576bdac3aa3aae0b6758c6efa115236d9e07dae44", size = 131099 },
    { url = "https://files.pythonhosted.org/packages/82/42/d1c7a7301d52af79d88548a97e297f9d99c961ad76bbe6f67442bb77f097/multidict-6.1.0-cp313-cp313-manylinux_2_5_i686.manylinux1_i686.manylinux_2_17_i686.manylinux2014_i686.whl", hash = "sha256:57feec87371dbb3520da6192213c7d6fc892d5589a93db548331954de8248fd2", size = 120403 },
    { url = "https://files.pythonhosted.org/packages/68/f3/471985c2c7ac707547553e8f37cff5158030d36bdec4414cb825fbaa5327/multidict-6.1.0-cp313-cp313-musllinux_1_2_aarch64.whl", hash = "sha256:0c3f390dc53279cbc8ba976e5f8035eab997829066756d811616b652b00a23a3", size = 125348 },
    { url = "https://files.pythonhosted.org/packages/67/2c/e6df05c77e0e433c214ec1d21ddd203d9a4770a1f2866a8ca40a545869a0/multidict-6.1.0-cp313-cp313-musllinux_1_2_i686.whl", hash = "sha256:59bfeae4b25ec05b34f1956eaa1cb38032282cd4dfabc5056d0a1ec4d696d3aa", size = 119673 },
    { url = "https://files.pythonhosted.org/packages/c5/cd/bc8608fff06239c9fb333f9db7743a1b2eafe98c2666c9a196e867a3a0a4/multidict-6.1.0-cp313-cp313-musllinux_1_2_ppc64le.whl", hash = "sha256:b2f59caeaf7632cc633b5cf6fc449372b83bbdf0da4ae04d5be36118e46cc0aa", size = 129927 },
    { url = "https://files.pythonhosted.org/packages/44/8e/281b69b7bc84fc963a44dc6e0bbcc7150e517b91df368a27834299a526ac/multidict-6.1.0-cp313-cp313-musllinux_1_2_s390x.whl", hash = "sha256:37bb93b2178e02b7b618893990941900fd25b6b9ac0fa49931a40aecdf083fe4", size = 128711 },
    { url = "https://files.pythonhosted.org/packages/12/a4/63e7cd38ed29dd9f1881d5119f272c898ca92536cdb53ffe0843197f6c85/multidict-6.1.0-cp313-cp313-musllinux_1_2_x86_64.whl", hash = "sha256:4e9f48f58c2c523d5a06faea47866cd35b32655c46b443f163d08c6d0ddb17d6", size = 125519 },
    { url = "https://files.pythonhosted.org/packages/38/e0/4f5855037a72cd8a7a2f60a3952d9aa45feedb37ae7831642102604e8a37/multidict-6.1.0-cp313-cp313-win32.whl", hash = "sha256:3a37ffb35399029b45c6cc33640a92bef403c9fd388acce75cdc88f58bd19a81", size = 26426 },
    { url = "https://files.pythonhosted.org/packages/7e/a5/17ee3a4db1e310b7405f5d25834460073a8ccd86198ce044dfaf69eac073/multidict-6.1.0-cp313-cp313-win_amd64.whl", hash = "sha256:e9aa71e15d9d9beaad2c6b9319edcdc0a49a43ef5c0a4c8265ca9ee7d6c67774", size = 28531 },
    { url = "https://files.pythonhosted.org/packages/99/b7/b9e70fde2c0f0c9af4cc5277782a89b66d35948ea3369ec9f598358c3ac5/multidict-6.1.0-py3-none-any.whl", hash = "sha256:48e171e52d1c4d33888e529b999e5900356b9ae588c2f09a52dcefb158b27506", size = 10051 },
]

[[package]]
name = "nest-asyncio"
version = "1.6.0"
source = { registry = "https://pypi.org/simple" }
sdist = { url = "https://files.pythonhosted.org/packages/83/f8/51569ac65d696c8ecbee95938f89d4abf00f47d58d48f6fbabfe8f0baefe/nest_asyncio-1.6.0.tar.gz", hash = "sha256:6f172d5449aca15afd6c646851f4e31e02c598d553a667e38cafa997cfec55fe", size = 7418 }
wheels = [
    { url = "https://files.pythonhosted.org/packages/a0/c4/c2971a3ba4c6103a3d10c4b0f24f461ddc027f0f09763220cf35ca1401b3/nest_asyncio-1.6.0-py3-none-any.whl", hash = "sha256:87af6efd6b5e897c81050477ef65c62e2b2f35d51703cae01aff2905b1852e1c", size = 5195 },
]

[[package]]
name = "numpy"
version = "2.1.2"
source = { registry = "https://pypi.org/simple" }
sdist = { url = "https://files.pythonhosted.org/packages/4b/d1/8a730ea07f4a37d94f9172f4ce1d81064b7a64766b460378be278952de75/numpy-2.1.2.tar.gz", hash = "sha256:13532a088217fa624c99b843eeb54640de23b3414b14aa66d023805eb731066c", size = 18878063 }
wheels = [
    { url = "https://files.pythonhosted.org/packages/aa/9c/9a6ec3ae89cd0648d419781284308f2956d2a61d932b5ac9682c956a171b/numpy-2.1.2-cp311-cp311-macosx_10_9_x86_64.whl", hash = "sha256:b42a1a511c81cc78cbc4539675713bbcf9d9c3913386243ceff0e9429ca892fe", size = 21154845 },
    { url = "https://files.pythonhosted.org/packages/02/69/9f05c4ecc75fabf297b17743996371b4c3dfc4d92e15c5c38d8bb3db8d74/numpy-2.1.2-cp311-cp311-macosx_11_0_arm64.whl", hash = "sha256:faa88bc527d0f097abdc2c663cddf37c05a1c2f113716601555249805cf573f1", size = 13789409 },
    { url = "https://files.pythonhosted.org/packages/34/4e/f95c99217bf77bbfaaf660d693c10bd0dc03b6032d19316d316088c9e479/numpy-2.1.2-cp311-cp311-macosx_14_0_arm64.whl", hash = "sha256:c82af4b2ddd2ee72d1fc0c6695048d457e00b3582ccde72d8a1c991b808bb20f", size = 5352097 },
    { url = "https://files.pythonhosted.org/packages/06/13/f5d87a497c16658e9af8920449b0b5692b469586b8231340c672962071c5/numpy-2.1.2-cp311-cp311-macosx_14_0_x86_64.whl", hash = "sha256:13602b3174432a35b16c4cfb5de9a12d229727c3dd47a6ce35111f2ebdf66ff4", size = 6891195 },
    { url = "https://files.pythonhosted.org/packages/6c/89/691ac07429ac061b344d5e37fa8e94be51a6017734aea15f2d9d7c6d119a/numpy-2.1.2-cp311-cp311-manylinux_2_17_aarch64.manylinux2014_aarch64.whl", hash = "sha256:1ebec5fd716c5a5b3d8dfcc439be82a8407b7b24b230d0ad28a81b61c2f4659a", size = 13895153 },
    { url = "https://files.pythonhosted.org/packages/23/69/538317f0d925095537745f12aced33be1570bbdc4acde49b33748669af96/numpy-2.1.2-cp311-cp311-manylinux_2_17_x86_64.manylinux2014_x86_64.whl", hash = "sha256:e2b49c3c0804e8ecb05d59af8386ec2f74877f7ca8fd9c1e00be2672e4d399b1", size = 16338306 },
    { url = "https://files.pythonhosted.org/packages/af/03/863fe7062c2106d3c151f7df9353f2ae2237c1dd6900f127a3eb1f24cb1b/numpy-2.1.2-cp311-cp311-musllinux_1_1_x86_64.whl", hash = "sha256:2cbba4b30bf31ddbe97f1c7205ef976909a93a66bb1583e983adbd155ba72ac2", size = 16710893 },
    { url = "https://files.pythonhosted.org/packages/70/77/0ad9efe25482009873f9660d29a40a8c41a6f0e8b541195e3c95c70684c5/numpy-2.1.2-cp311-cp311-musllinux_1_2_aarch64.whl", hash = "sha256:8e00ea6fc82e8a804433d3e9cedaa1051a1422cb6e443011590c14d2dea59146", size = 14398048 },
    { url = "https://files.pythonhosted.org/packages/3e/0f/e785fe75544db9f2b0bb1c181e13ceff349ce49753d807fd9672916aa06d/numpy-2.1.2-cp311-cp311-win32.whl", hash = "sha256:5006b13a06e0b38d561fab5ccc37581f23c9511879be7693bd33c7cd15ca227c", size = 6533458 },
    { url = "https://files.pythonhosted.org/packages/d4/96/450054662295125af861d48d2c4bc081dadcf1974a879b2104613157aa62/numpy-2.1.2-cp311-cp311-win_amd64.whl", hash = "sha256:f1eb068ead09f4994dec71c24b2844f1e4e4e013b9629f812f292f04bd1510d9", size = 12870896 },
    { url = "https://files.pythonhosted.org/packages/a0/7d/554a6838f37f3ada5a55f25173c619d556ae98092a6e01afb6e710501d70/numpy-2.1.2-cp312-cp312-macosx_10_13_x86_64.whl", hash = "sha256:d7bf0a4f9f15b32b5ba53147369e94296f5fffb783db5aacc1be15b4bf72f43b", size = 20848077 },
    { url = "https://files.pythonhosted.org/packages/b0/29/cb48a402ea879e645b16218718f3f7d9588a77d674a9dcf22e4c43487636/numpy-2.1.2-cp312-cp312-macosx_11_0_arm64.whl", hash = "sha256:b1d0fcae4f0949f215d4632be684a539859b295e2d0cb14f78ec231915d644db", size = 13493242 },
    { url = "https://files.pythonhosted.org/packages/56/44/f899b0581766c230da42f751b7b8896d096640b19b312164c267e48d36cb/numpy-2.1.2-cp312-cp312-macosx_14_0_arm64.whl", hash = "sha256:f751ed0a2f250541e19dfca9f1eafa31a392c71c832b6bb9e113b10d050cb0f1", size = 5089219 },
    { url = "https://files.pythonhosted.org/packages/79/8f/b987070d45161a7a4504afc67ed38544ed2c0ed5576263599a0402204a9c/numpy-2.1.2-cp312-cp312-macosx_14_0_x86_64.whl", hash = "sha256:bd33f82e95ba7ad632bc57837ee99dba3d7e006536200c4e9124089e1bf42426", size = 6620167 },
    { url = "https://files.pythonhosted.org/packages/c4/a7/af3329fda3c3ec31d9b650e42bbcd3422fc62a765cbb1405fde4177a0996/numpy-2.1.2-cp312-cp312-manylinux_2_17_aarch64.manylinux2014_aarch64.whl", hash = "sha256:1b8cde4f11f0a975d1fd59373b32e2f5a562ade7cde4f85b7137f3de8fbb29a0", size = 13604905 },
    { url = "https://files.pythonhosted.org/packages/9b/b4/e3c7e6fab0f77fff6194afa173d1f2342073d91b1d3b4b30b17c3fb4407a/numpy-2.1.2-cp312-cp312-manylinux_2_17_x86_64.manylinux2014_x86_64.whl", hash = "sha256:6d95f286b8244b3649b477ac066c6906fbb2905f8ac19b170e2175d3d799f4df", size = 16041825 },
    { url = "https://files.pythonhosted.org/packages/e9/50/6828e66a78aa03147c111f84d55f33ce2dde547cb578d6744a3b06a0124b/numpy-2.1.2-cp312-cp312-musllinux_1_1_x86_64.whl", hash = "sha256:ab4754d432e3ac42d33a269c8567413bdb541689b02d93788af4131018cbf366", size = 16409541 },
    { url = "https://files.pythonhosted.org/packages/bf/72/66af7916d9c3c6dbfbc8acdd4930c65461e1953374a2bc43d00f948f004a/numpy-2.1.2-cp312-cp312-musllinux_1_2_aarch64.whl", hash = "sha256:e585c8ae871fd38ac50598f4763d73ec5497b0de9a0ab4ef5b69f01c6a046142", size = 14081134 },
    { url = "https://files.pythonhosted.org/packages/dc/5a/59a67d84f33fe00ae74f0b5b69dd4f93a586a4aba7f7e19b54b2133db038/numpy-2.1.2-cp312-cp312-win32.whl", hash = "sha256:9c6c754df29ce6a89ed23afb25550d1c2d5fdb9901d9c67a16e0b16eaf7e2550", size = 6237784 },
    { url = "https://files.pythonhosted.org/packages/4c/79/73735a6a5dad6059c085f240a4e74c9270feccd2bc66e4d31b5ca01d329c/numpy-2.1.2-cp312-cp312-win_amd64.whl", hash = "sha256:456e3b11cb79ac9946c822a56346ec80275eaf2950314b249b512896c0d2505e", size = 12568254 },
    { url = "https://files.pythonhosted.org/packages/16/72/716fa1dbe92395a9a623d5049203ff8ddb0cfce65b9df9117c3696ccc011/numpy-2.1.2-cp313-cp313-macosx_10_13_x86_64.whl", hash = "sha256:a84498e0d0a1174f2b3ed769b67b656aa5460c92c9554039e11f20a05650f00d", size = 20834690 },
    { url = "https://files.pythonhosted.org/packages/1e/fb/3e85a39511586053b5c6a59a643879e376fae22230ebfef9cfabb0e032e2/numpy-2.1.2-cp313-cp313-macosx_11_0_arm64.whl", hash = "sha256:4d6ec0d4222e8ffdab1744da2560f07856421b367928026fb540e1945f2eeeaf", size = 13507474 },
    { url = "https://files.pythonhosted.org/packages/35/eb/5677556d9ba13436dab51e129f98d4829d95cd1b6bd0e199c14485a4bdb9/numpy-2.1.2-cp313-cp313-macosx_14_0_arm64.whl", hash = "sha256:259ec80d54999cc34cd1eb8ded513cb053c3bf4829152a2e00de2371bd406f5e", size = 5074742 },
    { url = "https://files.pythonhosted.org/packages/3e/c5/6c5ef5ba41b65a7e51bed50dbf3e1483eb578055633dd013e811a28e96a1/numpy-2.1.2-cp313-cp313-macosx_14_0_x86_64.whl", hash = "sha256:675c741d4739af2dc20cd6c6a5c4b7355c728167845e3c6b0e824e4e5d36a6c3", size = 6606787 },
    { url = "https://files.pythonhosted.org/packages/08/ac/f2f29dd4fd325b379c7dc932a0ebab22f0e031dbe80b2f6019b291a3a544/numpy-2.1.2-cp313-cp313-manylinux_2_17_aarch64.manylinux2014_aarch64.whl", hash = "sha256:05b2d4e667895cc55e3ff2b56077e4c8a5604361fc21a042845ea3ad67465aa8", size = 13601333 },
    { url = "https://files.pythonhosted.org/packages/44/26/63f5f4e5089654dfb858f4892215ed968cd1a68e6f4a83f9961f84f855cb/numpy-2.1.2-cp313-cp313-manylinux_2_17_x86_64.manylinux2014_x86_64.whl", hash = "sha256:43cca367bf94a14aca50b89e9bc2061683116cfe864e56740e083392f533ce7a", size = 16038090 },
    { url = "https://files.pythonhosted.org/packages/1d/21/015e0594de9c3a8d5edd24943d2bd23f102ec71aec026083f822f86497e2/numpy-2.1.2-cp313-cp313-musllinux_1_1_x86_64.whl", hash = "sha256:76322dcdb16fccf2ac56f99048af32259dcc488d9b7e25b51e5eca5147a3fb98", size = 16410865 },
    { url = "https://files.pythonhosted.org/packages/df/01/c1bcf9e6025d79077fbf3f3ee503b50aa7bfabfcd8f4b54f5829f4c00f3f/numpy-2.1.2-cp313-cp313-musllinux_1_2_aarch64.whl", hash = "sha256:32e16a03138cabe0cb28e1007ee82264296ac0983714094380b408097a418cfe", size = 14078077 },
    { url = "https://files.pythonhosted.org/packages/ba/06/db9d127d63bd11591770ba9f3d960f8041e0f895184b9351d4b1b5b56983/numpy-2.1.2-cp313-cp313-win32.whl", hash = "sha256:242b39d00e4944431a3cd2db2f5377e15b5785920421993770cddb89992c3f3a", size = 6234904 },
    { url = "https://files.pythonhosted.org/packages/a9/96/9f61f8f95b6e0ea0aa08633b704c75d1882bdcb331bdf8bfd63263b25b00/numpy-2.1.2-cp313-cp313-win_amd64.whl", hash = "sha256:f2ded8d9b6f68cc26f8425eda5d3877b47343e68ca23d0d0846f4d312ecaa445", size = 12561910 },
    { url = "https://files.pythonhosted.org/packages/36/b8/033f627821784a48e8f75c218033471eebbaacdd933f8979c79637a1b44b/numpy-2.1.2-cp313-cp313t-macosx_10_13_x86_64.whl", hash = "sha256:2ffef621c14ebb0188a8633348504a35c13680d6da93ab5cb86f4e54b7e922b5", size = 20857719 },
    { url = "https://files.pythonhosted.org/packages/96/46/af5726fde5b74ed83f2f17a73386d399319b7ed4d51279fb23b721d0816d/numpy-2.1.2-cp313-cp313t-macosx_11_0_arm64.whl", hash = "sha256:ad369ed238b1959dfbade9018a740fb9392c5ac4f9b5173f420bd4f37ba1f7a0", size = 13518826 },
    { url = "https://files.pythonhosted.org/packages/db/6e/8ce677edf36da1c4dae80afe5529f47690697eb55b4864673af260ccea7b/numpy-2.1.2-cp313-cp313t-macosx_14_0_arm64.whl", hash = "sha256:d82075752f40c0ddf57e6e02673a17f6cb0f8eb3f587f63ca1eaab5594da5b17", size = 5115036 },
    { url = "https://files.pythonhosted.org/packages/6a/ba/3cce44fb1b8438042c11847048812a776f75ee0e7070179c22e4cfbf420c/numpy-2.1.2-cp313-cp313t-macosx_14_0_x86_64.whl", hash = "sha256:1600068c262af1ca9580a527d43dc9d959b0b1d8e56f8a05d830eea39b7c8af6", size = 6628641 },
    { url = "https://files.pythonhosted.org/packages/59/c8/e722998720ccbd35ffbcf1d1b8ed0aa2304af88d3f1c38e06ebf983599b3/numpy-2.1.2-cp313-cp313t-manylinux_2_17_aarch64.manylinux2014_aarch64.whl", hash = "sha256:a26ae94658d3ba3781d5e103ac07a876b3e9b29db53f68ed7df432fd033358a8", size = 13574803 },
    { url = "https://files.pythonhosted.org/packages/7c/8e/fc1fdd83a55476765329ac2913321c4aed5b082a7915095628c4ca30ea72/numpy-2.1.2-cp313-cp313t-manylinux_2_17_x86_64.manylinux2014_x86_64.whl", hash = "sha256:13311c2db4c5f7609b462bc0f43d3c465424d25c626d95040f073e30f7570e35", size = 16021174 },
    { url = "https://files.pythonhosted.org/packages/2a/b6/a790742aa88067adb4bd6c89a946778c1417d4deaeafce3ca928f26d4c52/numpy-2.1.2-cp313-cp313t-musllinux_1_1_x86_64.whl", hash = "sha256:2abbf905a0b568706391ec6fa15161fad0fb5d8b68d73c461b3c1bab6064dd62", size = 16400117 },
    { url = "https://files.pythonhosted.org/packages/48/6f/129e3c17e3befe7fefdeaa6890f4c4df3f3cf0831aa053802c3862da67aa/numpy-2.1.2-cp313-cp313t-musllinux_1_2_aarch64.whl", hash = "sha256:ef444c57d664d35cac4e18c298c47d7b504c66b17c2ea91312e979fcfbdfb08a", size = 14066202 },
]

[[package]]
name = "oauthlib"
version = "3.2.2"
source = { registry = "https://pypi.org/simple" }
sdist = { url = "https://files.pythonhosted.org/packages/6d/fa/fbf4001037904031639e6bfbfc02badfc7e12f137a8afa254df6c4c8a670/oauthlib-3.2.2.tar.gz", hash = "sha256:9859c40929662bec5d64f34d01c99e093149682a3f38915dc0655d5a633dd918", size = 177352 }
wheels = [
    { url = "https://files.pythonhosted.org/packages/7e/80/cab10959dc1faead58dc8384a781dfbf93cb4d33d50988f7a69f1b7c9bbe/oauthlib-3.2.2-py3-none-any.whl", hash = "sha256:8139f29aac13e25d502680e9e19963e83f16838d48a0d71c287fe40e7067fbca", size = 151688 },
]

[[package]]
name = "openai"
version = "1.52.2"
source = { registry = "https://pypi.org/simple" }
dependencies = [
    { name = "anyio" },
    { name = "distro" },
    { name = "httpx" },
    { name = "jiter" },
    { name = "pydantic" },
    { name = "sniffio" },
    { name = "tqdm" },
    { name = "typing-extensions" },
]
sdist = { url = "https://files.pythonhosted.org/packages/a5/78/1c4658043cdbb7faf7f388cbb3902d5f8b9a307e10f2021b1a8a4b0b8b15/openai-1.52.2.tar.gz", hash = "sha256:87b7d0f69d85f5641678d414b7ee3082363647a5c66a462ed7f3ccb59582da0d", size = 310119 }
wheels = [
    { url = "https://files.pythonhosted.org/packages/55/4c/906b5b32c4c01402ac3b4c3fc28f601443ac5c6f13c84a95dd178c8d545d/openai-1.52.2-py3-none-any.whl", hash = "sha256:57e9e37bc407f39bb6ec3a27d7e8fb9728b2779936daa1fcf95df17d3edfaccc", size = 386947 },
]

[[package]]
name = "orjson"
version = "3.10.10"
source = { registry = "https://pypi.org/simple" }
sdist = { url = "https://files.pythonhosted.org/packages/80/44/d36e86b33fc84f224b5f2cdf525adf3b8f9f475753e721c402b1ddef731e/orjson-3.10.10.tar.gz", hash = "sha256:37949383c4df7b4337ce82ee35b6d7471e55195efa7dcb45ab8226ceadb0fe3b", size = 5404170 }
wheels = [
    { url = "https://files.pythonhosted.org/packages/79/bc/2a0eb0029729f1e466d5a595261446e5c5b6ed9213759ee56b6202f99417/orjson-3.10.10-cp311-cp311-macosx_10_15_x86_64.macosx_11_0_arm64.macosx_10_15_universal2.whl", hash = "sha256:879e99486c0fbb256266c7c6a67ff84f46035e4f8749ac6317cc83dacd7f993a", size = 270717 },
    { url = "https://files.pythonhosted.org/packages/3d/2b/5af226f183ce264bf64f15afe58647b09263dc1bde06aaadae6bbeca17f1/orjson-3.10.10-cp311-cp311-manylinux_2_17_aarch64.manylinux2014_aarch64.whl", hash = "sha256:019481fa9ea5ff13b5d5d95e6fd5ab25ded0810c80b150c2c7b1cc8660b662a7", size = 153294 },
    { url = "https://files.pythonhosted.org/packages/1d/95/d6a68ab51ed76e3794669dabb51bf7fa6ec2f4745f66e4af4518aeab4b73/orjson-3.10.10-cp311-cp311-manylinux_2_17_armv7l.manylinux2014_armv7l.whl", hash = "sha256:0dd57eff09894938b4c86d4b871a479260f9e156fa7f12f8cad4b39ea8028bb5", size = 168628 },
    { url = "https://files.pythonhosted.org/packages/c0/c9/1bbe5262f5e9df3e1aeec44ca8cc86846c7afb2746fa76bf668a7d0979e9/orjson-3.10.10-cp311-cp311-manylinux_2_17_ppc64le.manylinux2014_ppc64le.whl", hash = "sha256:dbde6d70cd95ab4d11ea8ac5e738e30764e510fc54d777336eec09bb93b8576c", size = 155845 },
    { url = "https://files.pythonhosted.org/packages/bf/22/e17b14ff74646e6c080dccb2859686a820bc6468f6b62ea3fe29a8bd3b05/orjson-3.10.10-cp311-cp311-manylinux_2_17_s390x.manylinux2014_s390x.whl", hash = "sha256:3b2625cb37b8fb42e2147404e5ff7ef08712099197a9cd38895006d7053e69d6", size = 166406 },
    { url = "https://files.pythonhosted.org/packages/8a/1e/b3abbe352f648f96a418acd1e602b1c77ffcc60cf801a57033da990b2c49/orjson-3.10.10-cp311-cp311-manylinux_2_17_x86_64.manylinux2014_x86_64.whl", hash = "sha256:dbf3c20c6a7db69df58672a0d5815647ecf78c8e62a4d9bd284e8621c1fe5ccb", size = 144518 },
    { url = "https://files.pythonhosted.org/packages/0e/5e/28f521ee0950d279489db1522e7a2460d0596df7c5ca452e242ff1509cfe/orjson-3.10.10-cp311-cp311-musllinux_1_2_aarch64.whl", hash = "sha256:75c38f5647e02d423807d252ce4528bf6a95bd776af999cb1fb48867ed01d1f6", size = 172187 },
    { url = "https://files.pythonhosted.org/packages/04/b4/538bf6f42eb0fd5a485abbe61e488d401a23fd6d6a758daefcf7811b6807/orjson-3.10.10-cp311-cp311-musllinux_1_2_x86_64.whl", hash = "sha256:23458d31fa50ec18e0ec4b0b4343730928296b11111df5f547c75913714116b2", size = 170152 },
    { url = "https://files.pythonhosted.org/packages/94/5c/a1a326a58452f9261972ad326ae3bb46d7945681239b7062a1b85d8811e2/orjson-3.10.10-cp311-none-win32.whl", hash = "sha256:2787cd9dedc591c989f3facd7e3e86508eafdc9536a26ec277699c0aa63c685b", size = 145116 },
    { url = "https://files.pythonhosted.org/packages/df/12/a02965df75f5a247091306d6cf40a77d20bf6c0490d0a5cb8719551ee815/orjson-3.10.10-cp311-none-win_amd64.whl", hash = "sha256:6514449d2c202a75183f807bc755167713297c69f1db57a89a1ef4a0170ee269", size = 139307 },
    { url = "https://files.pythonhosted.org/packages/21/c6/f1d2ec3ffe9d6a23a62af0477cd11dd2926762e0186a1fad8658a4f48117/orjson-3.10.10-cp312-cp312-macosx_10_15_x86_64.macosx_11_0_arm64.macosx_10_15_universal2.whl", hash = "sha256:8564f48f3620861f5ef1e080ce7cd122ee89d7d6dacf25fcae675ff63b4d6e05", size = 270801 },
    { url = "https://files.pythonhosted.org/packages/52/01/eba0226efaa4d4be8e44d9685750428503a3803648878fa5607100a74f81/orjson-3.10.10-cp312-cp312-manylinux_2_17_aarch64.manylinux2014_aarch64.whl", hash = "sha256:c5bf161a32b479034098c5b81f2608f09167ad2fa1c06abd4e527ea6bf4837a9", size = 153221 },
    { url = "https://files.pythonhosted.org/packages/da/4b/a705f9d3ae4786955ee0ac840b20960add357e612f1b0a54883d1811fe1a/orjson-3.10.10-cp312-cp312-manylinux_2_17_armv7l.manylinux2014_armv7l.whl", hash = "sha256:68b65c93617bcafa7f04b74ae8bc2cc214bd5cb45168a953256ff83015c6747d", size = 168590 },
    { url = "https://files.pythonhosted.org/packages/de/6c/eb405252e7d9ae9905a12bad582cfe37ef8ef18fdfee941549cb5834c7b2/orjson-3.10.10-cp312-cp312-manylinux_2_17_ppc64le.manylinux2014_ppc64le.whl", hash = "sha256:e8e28406f97fc2ea0c6150f4c1b6e8261453318930b334abc419214c82314f85", size = 156052 },
    { url = "https://files.pythonhosted.org/packages/9f/e7/65a0461574078a38f204575153524876350f0865162faa6e6e300ecaa199/orjson-3.10.10-cp312-cp312-manylinux_2_17_s390x.manylinux2014_s390x.whl", hash = "sha256:e4d0d9fe174cc7a5bdce2e6c378bcdb4c49b2bf522a8f996aa586020e1b96cee", size = 166562 },
    { url = "https://files.pythonhosted.org/packages/dd/99/85780be173e7014428859ba0211e6f2a8f8038ea6ebabe344b42d5daa277/orjson-3.10.10-cp312-cp312-manylinux_2_17_x86_64.manylinux2014_x86_64.whl", hash = "sha256:b3be81c42f1242cbed03cbb3973501fcaa2675a0af638f8be494eaf37143d999", size = 144892 },
    { url = "https://files.pythonhosted.org/packages/ed/c0/c7c42a2daeb262da417f70064746b700786ee0811b9a5821d9d37543b29d/orjson-3.10.10-cp312-cp312-musllinux_1_2_aarch64.whl", hash = "sha256:65f9886d3bae65be026219c0a5f32dbbe91a9e6272f56d092ab22561ad0ea33b", size = 172093 },
    { url = "https://files.pythonhosted.org/packages/ad/9b/be8b3d3aec42aa47f6058482ace0d2ca3023477a46643d766e96281d5d31/orjson-3.10.10-cp312-cp312-musllinux_1_2_x86_64.whl", hash = "sha256:730ed5350147db7beb23ddaf072f490329e90a1d059711d364b49fe352ec987b", size = 170424 },
    { url = "https://files.pythonhosted.org/packages/1b/15/a4cc61e23c39b9dec4620cb95817c83c84078be1771d602f6d03f0e5c696/orjson-3.10.10-cp312-none-win32.whl", hash = "sha256:a8f4bf5f1c85bea2170800020d53a8877812892697f9c2de73d576c9307a8a5f", size = 145132 },
    { url = "https://files.pythonhosted.org/packages/9f/8a/ce7c28e4ea337f6d95261345d7c61322f8561c52f57b263a3ad7025984f4/orjson-3.10.10-cp312-none-win_amd64.whl", hash = "sha256:384cd13579a1b4cd689d218e329f459eb9ddc504fa48c5a83ef4889db7fd7a4f", size = 139389 },
    { url = "https://files.pythonhosted.org/packages/0c/69/f1c4382cd44bdaf10006c4e82cb85d2bcae735369f84031e203c4e5d87de/orjson-3.10.10-cp313-cp313-macosx_10_15_x86_64.macosx_11_0_arm64.macosx_10_15_universal2.whl", hash = "sha256:44bffae68c291f94ff5a9b4149fe9d1bdd4cd0ff0fb575bcea8351d48db629a1", size = 270695 },
    { url = "https://files.pythonhosted.org/packages/61/29/aeb5153271d4953872b06ed239eb54993a5f344353727c42d3aabb2046f6/orjson-3.10.10-cp313-cp313-manylinux_2_17_aarch64.manylinux2014_aarch64.whl", hash = "sha256:e27b4c6437315df3024f0835887127dac2a0a3ff643500ec27088d2588fa5ae1", size = 141632 },
    { url = "https://files.pythonhosted.org/packages/bc/a2/c8ac38d8fb461a9b717c766fbe1f7d3acf9bde2f12488eb13194960782e4/orjson-3.10.10-cp313-cp313-manylinux_2_17_x86_64.manylinux2014_x86_64.whl", hash = "sha256:bca84df16d6b49325a4084fd8b2fe2229cb415e15c46c529f868c3387bb1339d", size = 144854 },
    { url = "https://files.pythonhosted.org/packages/79/51/e7698fdb28bdec633888cc667edc29fd5376fce9ade0a5b3e22f5ebe0343/orjson-3.10.10-cp313-cp313-musllinux_1_2_aarch64.whl", hash = "sha256:c14ce70e8f39bd71f9f80423801b5d10bf93d1dceffdecd04df0f64d2c69bc01", size = 172023 },
    { url = "https://files.pythonhosted.org/packages/02/2d/0d99c20878658c7e33b90e6a4bb75cf2924d6ff29c2365262cff3c26589a/orjson-3.10.10-cp313-cp313-musllinux_1_2_x86_64.whl", hash = "sha256:24ac62336da9bda1bd93c0491eff0613003b48d3cb5d01470842e7b52a40d5b4", size = 170429 },
    { url = "https://files.pythonhosted.org/packages/cd/45/6a4a446f4fb29bb4703c3537d5c6a2bf7fed768cb4d7b7dce9d71b72fc93/orjson-3.10.10-cp313-none-win32.whl", hash = "sha256:eb0a42831372ec2b05acc9ee45af77bcaccbd91257345f93780a8e654efc75db", size = 145099 },
    { url = "https://files.pythonhosted.org/packages/72/6e/4631fe219a4203aa111e9bb763ad2e2e0cdd1a03805029e4da124d96863f/orjson-3.10.10-cp313-none-win_amd64.whl", hash = "sha256:f0c4f37f8bf3f1075c6cc8dd8a9f843689a4b618628f8812d0a71e6968b95ffd", size = 139176 },
]

[[package]]
name = "parsimonious"
version = "0.10.0"
source = { registry = "https://pypi.org/simple" }
dependencies = [
    { name = "regex" },
]
sdist = { url = "https://files.pythonhosted.org/packages/7b/91/abdc50c4ef06fdf8d047f60ee777ca9b2a7885e1a9cea81343fbecda52d7/parsimonious-0.10.0.tar.gz", hash = "sha256:8281600da180ec8ae35427a4ab4f7b82bfec1e3d1e52f80cb60ea82b9512501c", size = 52172 }
wheels = [
    { url = "https://files.pythonhosted.org/packages/aa/0f/c8b64d9b54ea631fcad4e9e3c8dbe8c11bb32a623be94f22974c88e71eaf/parsimonious-0.10.0-py3-none-any.whl", hash = "sha256:982ab435fabe86519b57f6b35610aa4e4e977e9f02a14353edf4bbc75369fc0f", size = 48427 },
]

[[package]]
name = "propcache"
version = "0.2.0"
source = { registry = "https://pypi.org/simple" }
sdist = { url = "https://files.pythonhosted.org/packages/a9/4d/5e5a60b78dbc1d464f8a7bbaeb30957257afdc8512cbb9dfd5659304f5cd/propcache-0.2.0.tar.gz", hash = "sha256:df81779732feb9d01e5d513fad0122efb3d53bbc75f61b2a4f29a020bc985e70", size = 40951 }
wheels = [
    { url = "https://files.pythonhosted.org/packages/e0/1c/71eec730e12aec6511e702ad0cd73c2872eccb7cad39de8ba3ba9de693ef/propcache-0.2.0-cp311-cp311-macosx_10_9_universal2.whl", hash = "sha256:63f13bf09cc3336eb04a837490b8f332e0db41da66995c9fd1ba04552e516354", size = 80811 },
    { url = "https://files.pythonhosted.org/packages/89/c3/7e94009f9a4934c48a371632197406a8860b9f08e3f7f7d922ab69e57a41/propcache-0.2.0-cp311-cp311-macosx_10_9_x86_64.whl", hash = "sha256:608cce1da6f2672a56b24a015b42db4ac612ee709f3d29f27a00c943d9e851de", size = 46365 },
    { url = "https://files.pythonhosted.org/packages/c0/1d/c700d16d1d6903aeab28372fe9999762f074b80b96a0ccc953175b858743/propcache-0.2.0-cp311-cp311-macosx_11_0_arm64.whl", hash = "sha256:466c219deee4536fbc83c08d09115249db301550625c7fef1c5563a584c9bc87", size = 45602 },
    { url = "https://files.pythonhosted.org/packages/2e/5e/4a3e96380805bf742712e39a4534689f4cddf5fa2d3a93f22e9fd8001b23/propcache-0.2.0-cp311-cp311-manylinux_2_17_aarch64.manylinux2014_aarch64.whl", hash = "sha256:fc2db02409338bf36590aa985a461b2c96fce91f8e7e0f14c50c5fcc4f229016", size = 236161 },
    { url = "https://files.pythonhosted.org/packages/a5/85/90132481183d1436dff6e29f4fa81b891afb6cb89a7306f32ac500a25932/propcache-0.2.0-cp311-cp311-manylinux_2_17_ppc64le.manylinux2014_ppc64le.whl", hash = "sha256:a6ed8db0a556343d566a5c124ee483ae113acc9a557a807d439bcecc44e7dfbb", size = 244938 },
    { url = "https://files.pythonhosted.org/packages/4a/89/c893533cb45c79c970834274e2d0f6d64383ec740be631b6a0a1d2b4ddc0/propcache-0.2.0-cp311-cp311-manylinux_2_17_s390x.manylinux2014_s390x.whl", hash = "sha256:91997d9cb4a325b60d4e3f20967f8eb08dfcb32b22554d5ef78e6fd1dda743a2", size = 243576 },
    { url = "https://files.pythonhosted.org/packages/8c/56/98c2054c8526331a05f205bf45cbb2cda4e58e56df70e76d6a509e5d6ec6/propcache-0.2.0-cp311-cp311-manylinux_2_17_x86_64.manylinux2014_x86_64.whl", hash = "sha256:4c7dde9e533c0a49d802b4f3f218fa9ad0a1ce21f2c2eb80d5216565202acab4", size = 236011 },
    { url = "https://files.pythonhosted.org/packages/2d/0c/8b8b9f8a6e1abd869c0fa79b907228e7abb966919047d294ef5df0d136cf/propcache-0.2.0-cp311-cp311-manylinux_2_5_i686.manylinux1_i686.manylinux_2_17_i686.manylinux2014_i686.whl", hash = "sha256:ffcad6c564fe6b9b8916c1aefbb37a362deebf9394bd2974e9d84232e3e08504", size = 224834 },
    { url = "https://files.pythonhosted.org/packages/18/bb/397d05a7298b7711b90e13108db697732325cafdcd8484c894885c1bf109/propcache-0.2.0-cp311-cp311-musllinux_1_2_aarch64.whl", hash = "sha256:97a58a28bcf63284e8b4d7b460cbee1edaab24634e82059c7b8c09e65284f178", size = 224946 },
    { url = "https://files.pythonhosted.org/packages/25/19/4fc08dac19297ac58135c03770b42377be211622fd0147f015f78d47cd31/propcache-0.2.0-cp311-cp311-musllinux_1_2_armv7l.whl", hash = "sha256:945db8ee295d3af9dbdbb698cce9bbc5c59b5c3fe328bbc4387f59a8a35f998d", size = 217280 },
    { url = "https://files.pythonhosted.org/packages/7e/76/c79276a43df2096ce2aba07ce47576832b1174c0c480fe6b04bd70120e59/propcache-0.2.0-cp311-cp311-musllinux_1_2_i686.whl", hash = "sha256:39e104da444a34830751715f45ef9fc537475ba21b7f1f5b0f4d71a3b60d7fe2", size = 220088 },
    { url = "https://files.pythonhosted.org/packages/c3/9a/8a8cf428a91b1336b883f09c8b884e1734c87f724d74b917129a24fe2093/propcache-0.2.0-cp311-cp311-musllinux_1_2_ppc64le.whl", hash = "sha256:c5ecca8f9bab618340c8e848d340baf68bcd8ad90a8ecd7a4524a81c1764b3db", size = 233008 },
    { url = "https://files.pythonhosted.org/packages/25/7b/768a8969abd447d5f0f3333df85c6a5d94982a1bc9a89c53c154bf7a8b11/propcache-0.2.0-cp311-cp311-musllinux_1_2_s390x.whl", hash = "sha256:c436130cc779806bdf5d5fae0d848713105472b8566b75ff70048c47d3961c5b", size = 237719 },
    { url = "https://files.pythonhosted.org/packages/ed/0d/e5d68ccc7976ef8b57d80613ac07bbaf0614d43f4750cf953f0168ef114f/propcache-0.2.0-cp311-cp311-musllinux_1_2_x86_64.whl", hash = "sha256:191db28dc6dcd29d1a3e063c3be0b40688ed76434622c53a284e5427565bbd9b", size = 227729 },
    { url = "https://files.pythonhosted.org/packages/05/64/17eb2796e2d1c3d0c431dc5f40078d7282f4645af0bb4da9097fbb628c6c/propcache-0.2.0-cp311-cp311-win32.whl", hash = "sha256:5f2564ec89058ee7c7989a7b719115bdfe2a2fb8e7a4543b8d1c0cc4cf6478c1", size = 40473 },
    { url = "https://files.pythonhosted.org/packages/83/c5/e89fc428ccdc897ade08cd7605f174c69390147526627a7650fb883e0cd0/propcache-0.2.0-cp311-cp311-win_amd64.whl", hash = "sha256:6e2e54267980349b723cff366d1e29b138b9a60fa376664a157a342689553f71", size = 44921 },
    { url = "https://files.pythonhosted.org/packages/7c/46/a41ca1097769fc548fc9216ec4c1471b772cc39720eb47ed7e38ef0006a9/propcache-0.2.0-cp312-cp312-macosx_10_13_universal2.whl", hash = "sha256:2ee7606193fb267be4b2e3b32714f2d58cad27217638db98a60f9efb5efeccc2", size = 80800 },
    { url = "https://files.pythonhosted.org/packages/75/4f/93df46aab9cc473498ff56be39b5f6ee1e33529223d7a4d8c0a6101a9ba2/propcache-0.2.0-cp312-cp312-macosx_10_13_x86_64.whl", hash = "sha256:91ee8fc02ca52e24bcb77b234f22afc03288e1dafbb1f88fe24db308910c4ac7", size = 46443 },
    { url = "https://files.pythonhosted.org/packages/0b/17/308acc6aee65d0f9a8375e36c4807ac6605d1f38074b1581bd4042b9fb37/propcache-0.2.0-cp312-cp312-macosx_11_0_arm64.whl", hash = "sha256:2e900bad2a8456d00a113cad8c13343f3b1f327534e3589acc2219729237a2e8", size = 45676 },
    { url = "https://files.pythonhosted.org/packages/65/44/626599d2854d6c1d4530b9a05e7ff2ee22b790358334b475ed7c89f7d625/propcache-0.2.0-cp312-cp312-manylinux_2_17_aarch64.manylinux2014_aarch64.whl", hash = "sha256:f52a68c21363c45297aca15561812d542f8fc683c85201df0bebe209e349f793", size = 246191 },
    { url = "https://files.pythonhosted.org/packages/f2/df/5d996d7cb18df076debae7d76ac3da085c0575a9f2be6b1f707fe227b54c/propcache-0.2.0-cp312-cp312-manylinux_2_17_ppc64le.manylinux2014_ppc64le.whl", hash = "sha256:1e41d67757ff4fbc8ef2af99b338bfb955010444b92929e9e55a6d4dcc3c4f09", size = 251791 },
    { url = "https://files.pythonhosted.org/packages/2e/6d/9f91e5dde8b1f662f6dd4dff36098ed22a1ef4e08e1316f05f4758f1576c/propcache-0.2.0-cp312-cp312-manylinux_2_17_s390x.manylinux2014_s390x.whl", hash = "sha256:a64e32f8bd94c105cc27f42d3b658902b5bcc947ece3c8fe7bc1b05982f60e89", size = 253434 },
    { url = "https://files.pythonhosted.org/packages/3c/e9/1b54b7e26f50b3e0497cd13d3483d781d284452c2c50dd2a615a92a087a3/propcache-0.2.0-cp312-cp312-manylinux_2_17_x86_64.manylinux2014_x86_64.whl", hash = "sha256:55346705687dbd7ef0d77883ab4f6fabc48232f587925bdaf95219bae072491e", size = 248150 },
    { url = "https://files.pythonhosted.org/packages/a7/ef/a35bf191c8038fe3ce9a414b907371c81d102384eda5dbafe6f4dce0cf9b/propcache-0.2.0-cp312-cp312-manylinux_2_5_i686.manylinux1_i686.manylinux_2_17_i686.manylinux2014_i686.whl", hash = "sha256:00181262b17e517df2cd85656fcd6b4e70946fe62cd625b9d74ac9977b64d8d9", size = 233568 },
    { url = "https://files.pythonhosted.org/packages/97/d9/d00bb9277a9165a5e6d60f2142cd1a38a750045c9c12e47ae087f686d781/propcache-0.2.0-cp312-cp312-musllinux_1_2_aarch64.whl", hash = "sha256:6994984550eaf25dd7fc7bd1b700ff45c894149341725bb4edc67f0ffa94efa4", size = 229874 },
    { url = "https://files.pythonhosted.org/packages/8e/78/c123cf22469bdc4b18efb78893e69c70a8b16de88e6160b69ca6bdd88b5d/propcache-0.2.0-cp312-cp312-musllinux_1_2_armv7l.whl", hash = "sha256:56295eb1e5f3aecd516d91b00cfd8bf3a13991de5a479df9e27dd569ea23959c", size = 225857 },
    { url = "https://files.pythonhosted.org/packages/31/1b/fd6b2f1f36d028820d35475be78859d8c89c8f091ad30e377ac49fd66359/propcache-0.2.0-cp312-cp312-musllinux_1_2_i686.whl", hash = "sha256:439e76255daa0f8151d3cb325f6dd4a3e93043e6403e6491813bcaaaa8733887", size = 227604 },
    { url = "https://files.pythonhosted.org/packages/99/36/b07be976edf77a07233ba712e53262937625af02154353171716894a86a6/propcache-0.2.0-cp312-cp312-musllinux_1_2_ppc64le.whl", hash = "sha256:f6475a1b2ecb310c98c28d271a30df74f9dd436ee46d09236a6b750a7599ce57", size = 238430 },
    { url = "https://files.pythonhosted.org/packages/0d/64/5822f496c9010e3966e934a011ac08cac8734561842bc7c1f65586e0683c/propcache-0.2.0-cp312-cp312-musllinux_1_2_s390x.whl", hash = "sha256:3444cdba6628accf384e349014084b1cacd866fbb88433cd9d279d90a54e0b23", size = 244814 },
    { url = "https://files.pythonhosted.org/packages/fd/bd/8657918a35d50b18a9e4d78a5df7b6c82a637a311ab20851eef4326305c1/propcache-0.2.0-cp312-cp312-musllinux_1_2_x86_64.whl", hash = "sha256:4a9d9b4d0a9b38d1c391bb4ad24aa65f306c6f01b512e10a8a34a2dc5675d348", size = 235922 },
    { url = "https://files.pythonhosted.org/packages/a8/6f/ec0095e1647b4727db945213a9f395b1103c442ef65e54c62e92a72a3f75/propcache-0.2.0-cp312-cp312-win32.whl", hash = "sha256:69d3a98eebae99a420d4b28756c8ce6ea5a29291baf2dc9ff9414b42676f61d5", size = 40177 },
    { url = "https://files.pythonhosted.org/packages/20/a2/bd0896fdc4f4c1db46d9bc361c8c79a9bf08ccc08ba054a98e38e7ba1557/propcache-0.2.0-cp312-cp312-win_amd64.whl", hash = "sha256:ad9c9b99b05f163109466638bd30ada1722abb01bbb85c739c50b6dc11f92dc3", size = 44446 },
    { url = "https://files.pythonhosted.org/packages/a8/a7/5f37b69197d4f558bfef5b4bceaff7c43cc9b51adf5bd75e9081d7ea80e4/propcache-0.2.0-cp313-cp313-macosx_10_13_universal2.whl", hash = "sha256:ecddc221a077a8132cf7c747d5352a15ed763b674c0448d811f408bf803d9ad7", size = 78120 },
    { url = "https://files.pythonhosted.org/packages/c8/cd/48ab2b30a6b353ecb95a244915f85756d74f815862eb2ecc7a518d565b48/propcache-0.2.0-cp313-cp313-macosx_10_13_x86_64.whl", hash = "sha256:0e53cb83fdd61cbd67202735e6a6687a7b491c8742dfc39c9e01e80354956763", size = 45127 },
    { url = "https://files.pythonhosted.org/packages/a5/ba/0a1ef94a3412aab057bd996ed5f0ac7458be5bf469e85c70fa9ceb43290b/propcache-0.2.0-cp313-cp313-macosx_11_0_arm64.whl", hash = "sha256:92fe151145a990c22cbccf9ae15cae8ae9eddabfc949a219c9f667877e40853d", size = 44419 },
    { url = "https://files.pythonhosted.org/packages/b4/6c/ca70bee4f22fa99eacd04f4d2f1699be9d13538ccf22b3169a61c60a27fa/propcache-0.2.0-cp313-cp313-manylinux_2_17_aarch64.manylinux2014_aarch64.whl", hash = "sha256:d6a21ef516d36909931a2967621eecb256018aeb11fc48656e3257e73e2e247a", size = 229611 },
    { url = "https://files.pythonhosted.org/packages/19/70/47b872a263e8511ca33718d96a10c17d3c853aefadeb86dc26e8421184b9/propcache-0.2.0-cp313-cp313-manylinux_2_17_ppc64le.manylinux2014_ppc64le.whl", hash = "sha256:3f88a4095e913f98988f5b338c1d4d5d07dbb0b6bad19892fd447484e483ba6b", size = 234005 },
    { url = "https://files.pythonhosted.org/packages/4f/be/3b0ab8c84a22e4a3224719099c1229ddfdd8a6a1558cf75cb55ee1e35c25/propcache-0.2.0-cp313-cp313-manylinux_2_17_s390x.manylinux2014_s390x.whl", hash = "sha256:5a5b3bb545ead161be780ee85a2b54fdf7092815995661947812dde94a40f6fb", size = 237270 },
    { url = "https://files.pythonhosted.org/packages/04/d8/f071bb000d4b8f851d312c3c75701e586b3f643fe14a2e3409b1b9ab3936/propcache-0.2.0-cp313-cp313-manylinux_2_17_x86_64.manylinux2014_x86_64.whl", hash = "sha256:67aeb72e0f482709991aa91345a831d0b707d16b0257e8ef88a2ad246a7280bf", size = 231877 },
    { url = "https://files.pythonhosted.org/packages/93/e7/57a035a1359e542bbb0a7df95aad6b9871ebee6dce2840cb157a415bd1f3/propcache-0.2.0-cp313-cp313-manylinux_2_5_i686.manylinux1_i686.manylinux_2_17_i686.manylinux2014_i686.whl", hash = "sha256:3c997f8c44ec9b9b0bcbf2d422cc00a1d9b9c681f56efa6ca149a941e5560da2", size = 217848 },
    { url = "https://files.pythonhosted.org/packages/f0/93/d1dea40f112ec183398fb6c42fde340edd7bab202411c4aa1a8289f461b6/propcache-0.2.0-cp313-cp313-musllinux_1_2_aarch64.whl", hash = "sha256:2a66df3d4992bc1d725b9aa803e8c5a66c010c65c741ad901e260ece77f58d2f", size = 216987 },
    { url = "https://files.pythonhosted.org/packages/62/4c/877340871251145d3522c2b5d25c16a1690ad655fbab7bb9ece6b117e39f/propcache-0.2.0-cp313-cp313-musllinux_1_2_armv7l.whl", hash = "sha256:3ebbcf2a07621f29638799828b8d8668c421bfb94c6cb04269130d8de4fb7136", size = 212451 },
    { url = "https://files.pythonhosted.org/packages/7c/bb/a91b72efeeb42906ef58ccf0cdb87947b54d7475fee3c93425d732f16a61/propcache-0.2.0-cp313-cp313-musllinux_1_2_i686.whl", hash = "sha256:1235c01ddaa80da8235741e80815ce381c5267f96cc49b1477fdcf8c047ef325", size = 212879 },
    { url = "https://files.pythonhosted.org/packages/9b/7f/ee7fea8faac57b3ec5d91ff47470c6c5d40d7f15d0b1fccac806348fa59e/propcache-0.2.0-cp313-cp313-musllinux_1_2_ppc64le.whl", hash = "sha256:3947483a381259c06921612550867b37d22e1df6d6d7e8361264b6d037595f44", size = 222288 },
    { url = "https://files.pythonhosted.org/packages/ff/d7/acd67901c43d2e6b20a7a973d9d5fd543c6e277af29b1eb0e1f7bd7ca7d2/propcache-0.2.0-cp313-cp313-musllinux_1_2_s390x.whl", hash = "sha256:d5bed7f9805cc29c780f3aee05de3262ee7ce1f47083cfe9f77471e9d6777e83", size = 228257 },
    { url = "https://files.pythonhosted.org/packages/8d/6f/6272ecc7a8daad1d0754cfc6c8846076a8cb13f810005c79b15ce0ef0cf2/propcache-0.2.0-cp313-cp313-musllinux_1_2_x86_64.whl", hash = "sha256:e4a91d44379f45f5e540971d41e4626dacd7f01004826a18cb048e7da7e96544", size = 221075 },
    { url = "https://files.pythonhosted.org/packages/7c/bd/c7a6a719a6b3dd8b3aeadb3675b5783983529e4a3185946aa444d3e078f6/propcache-0.2.0-cp313-cp313-win32.whl", hash = "sha256:f902804113e032e2cdf8c71015651c97af6418363bea8d78dc0911d56c335032", size = 39654 },
    { url = "https://files.pythonhosted.org/packages/88/e7/0eef39eff84fa3e001b44de0bd41c7c0e3432e7648ffd3d64955910f002d/propcache-0.2.0-cp313-cp313-win_amd64.whl", hash = "sha256:8f188cfcc64fb1266f4684206c9de0e80f54622c3f22a910cbd200478aeae61e", size = 43705 },
    { url = "https://files.pythonhosted.org/packages/3d/b6/e6d98278f2d49b22b4d033c9f792eda783b9ab2094b041f013fc69bcde87/propcache-0.2.0-py3-none-any.whl", hash = "sha256:2ccc28197af5313706511fab3a8b66dcd6da067a1331372c82ea1cb74285e036", size = 11603 },
]

[[package]]
name = "pycryptodome"
version = "3.21.0"
source = { registry = "https://pypi.org/simple" }
sdist = { url = "https://files.pythonhosted.org/packages/13/52/13b9db4a913eee948152a079fe58d035bd3d1a519584155da8e786f767e6/pycryptodome-3.21.0.tar.gz", hash = "sha256:f7787e0d469bdae763b876174cf2e6c0f7be79808af26b1da96f1a64bcf47297", size = 4818071 }
wheels = [
    { url = "https://files.pythonhosted.org/packages/a7/88/5e83de10450027c96c79dc65ac45e9d0d7a7fef334f39d3789a191f33602/pycryptodome-3.21.0-cp36-abi3-macosx_10_9_universal2.whl", hash = "sha256:2480ec2c72438430da9f601ebc12c518c093c13111a5c1644c82cdfc2e50b1e4", size = 2495937 },
    { url = "https://files.pythonhosted.org/packages/66/e1/8f28cd8cf7f7563319819d1e172879ccce2333781ae38da61c28fe22d6ff/pycryptodome-3.21.0-cp36-abi3-macosx_10_9_x86_64.whl", hash = "sha256:de18954104667f565e2fbb4783b56667f30fb49c4d79b346f52a29cb198d5b6b", size = 1634629 },
    { url = "https://files.pythonhosted.org/packages/6a/c1/f75a1aaff0c20c11df8dc8e2bf8057e7f73296af7dfd8cbb40077d1c930d/pycryptodome-3.21.0-cp36-abi3-manylinux_2_17_aarch64.manylinux2014_aarch64.whl", hash = "sha256:2de4b7263a33947ff440412339cb72b28a5a4c769b5c1ca19e33dd6cd1dcec6e", size = 2168708 },
    { url = "https://files.pythonhosted.org/packages/ea/66/6f2b7ddb457b19f73b82053ecc83ba768680609d56dd457dbc7e902c41aa/pycryptodome-3.21.0-cp36-abi3-manylinux_2_17_x86_64.manylinux2014_x86_64.whl", hash = "sha256:0714206d467fc911042d01ea3a1847c847bc10884cf674c82e12915cfe1649f8", size = 2254555 },
    { url = "https://files.pythonhosted.org/packages/2c/2b/152c330732a887a86cbf591ed69bd1b489439b5464806adb270f169ec139/pycryptodome-3.21.0-cp36-abi3-manylinux_2_5_i686.manylinux1_i686.manylinux_2_17_i686.manylinux2014_i686.whl", hash = "sha256:7d85c1b613121ed3dbaa5a97369b3b757909531a959d229406a75b912dd51dd1", size = 2294143 },
    { url = "https://files.pythonhosted.org/packages/55/92/517c5c498c2980c1b6d6b9965dffbe31f3cd7f20f40d00ec4069559c5902/pycryptodome-3.21.0-cp36-abi3-musllinux_1_1_aarch64.whl", hash = "sha256:8898a66425a57bcf15e25fc19c12490b87bd939800f39a03ea2de2aea5e3611a", size = 2160509 },
    { url = "https://files.pythonhosted.org/packages/39/1f/c74288f54d80a20a78da87df1818c6464ac1041d10988bb7d982c4153fbc/pycryptodome-3.21.0-cp36-abi3-musllinux_1_2_i686.whl", hash = "sha256:932c905b71a56474bff8a9c014030bc3c882cee696b448af920399f730a650c2", size = 2329480 },
    { url = "https://files.pythonhosted.org/packages/39/1b/d0b013bf7d1af7cf0a6a4fce13f5fe5813ab225313755367b36e714a63f8/pycryptodome-3.21.0-cp36-abi3-musllinux_1_2_x86_64.whl", hash = "sha256:18caa8cfbc676eaaf28613637a89980ad2fd96e00c564135bf90bc3f0b34dd93", size = 2254397 },
    { url = "https://files.pythonhosted.org/packages/14/71/4cbd3870d3e926c34706f705d6793159ac49d9a213e3ababcdade5864663/pycryptodome-3.21.0-cp36-abi3-win32.whl", hash = "sha256:280b67d20e33bb63171d55b1067f61fbd932e0b1ad976b3a184303a3dad22764", size = 1775641 },
    { url = "https://files.pythonhosted.org/packages/43/1d/81d59d228381576b92ecede5cd7239762c14001a828bdba30d64896e9778/pycryptodome-3.21.0-cp36-abi3-win_amd64.whl", hash = "sha256:b7aa25fc0baa5b1d95b7633af4f5f1838467f1815442b22487426f94e0d66c53", size = 1812863 },
    { url = "https://files.pythonhosted.org/packages/25/b3/09ff7072e6d96c9939c24cf51d3c389d7c345bf675420355c22402f71b68/pycryptodome-3.21.0-pp27-pypy_73-manylinux2010_x86_64.whl", hash = "sha256:2cb635b67011bc147c257e61ce864879ffe6d03342dc74b6045059dfbdedafca", size = 1691593 },
    { url = "https://files.pythonhosted.org/packages/a8/91/38e43628148f68ba9b68dedbc323cf409e537fd11264031961fd7c744034/pycryptodome-3.21.0-pp27-pypy_73-win32.whl", hash = "sha256:4c26a2f0dc15f81ea3afa3b0c87b87e501f235d332b7f27e2225ecb80c0b1cdd", size = 1765997 },
]

[[package]]
name = "pydantic"
version = "2.8.2"
source = { registry = "https://pypi.org/simple" }
dependencies = [
    { name = "annotated-types" },
    { name = "pydantic-core" },
    { name = "typing-extensions" },
]
sdist = { url = "https://files.pythonhosted.org/packages/8c/99/d0a5dca411e0a017762258013ba9905cd6e7baa9a3fd1fe8b6529472902e/pydantic-2.8.2.tar.gz", hash = "sha256:6f62c13d067b0755ad1c21a34bdd06c0c12625a22b0fc09c6b149816604f7c2a", size = 739834 }
wheels = [
    { url = "https://files.pythonhosted.org/packages/1f/fa/b7f815b8c9ad021c07f88875b601222ef5e70619391ade4a49234d12d278/pydantic-2.8.2-py3-none-any.whl", hash = "sha256:73ee9fddd406dc318b885c7a2eab8a6472b68b8fb5ba8150949fc3db939f23c8", size = 423875 },
]

[[package]]
name = "pydantic-core"
version = "2.20.1"
source = { registry = "https://pypi.org/simple" }
dependencies = [
    { name = "typing-extensions" },
]
sdist = { url = "https://files.pythonhosted.org/packages/12/e3/0d5ad91211dba310f7ded335f4dad871172b9cc9ce204f5a56d76ccd6247/pydantic_core-2.20.1.tar.gz", hash = "sha256:26ca695eeee5f9f1aeeb211ffc12f10bcb6f71e2989988fda61dabd65db878d4", size = 388371 }
wheels = [
    { url = "https://files.pythonhosted.org/packages/61/db/f6a724db226d990a329910727cfac43539ff6969edc217286dd05cda3ef6/pydantic_core-2.20.1-cp311-cp311-macosx_10_12_x86_64.whl", hash = "sha256:d2a8fa9d6d6f891f3deec72f5cc668e6f66b188ab14bb1ab52422fe8e644f312", size = 1834507 },
    { url = "https://files.pythonhosted.org/packages/9b/83/6f2bfe75209d557ae1c3550c1252684fc1827b8b12fbed84c3b4439e135d/pydantic_core-2.20.1-cp311-cp311-macosx_11_0_arm64.whl", hash = "sha256:175873691124f3d0da55aeea1d90660a6ea7a3cfea137c38afa0a5ffabe37b88", size = 1773527 },
    { url = "https://files.pythonhosted.org/packages/93/ef/513ea76d7ca81f2354bb9c8d7839fc1157673e652613f7e1aff17d8ce05d/pydantic_core-2.20.1-cp311-cp311-manylinux_2_17_aarch64.manylinux2014_aarch64.whl", hash = "sha256:37eee5b638f0e0dcd18d21f59b679686bbd18917b87db0193ae36f9c23c355fc", size = 1787879 },
    { url = "https://files.pythonhosted.org/packages/31/0a/ac294caecf235f0cc651de6232f1642bb793af448d1cfc541b0dc1fd72b8/pydantic_core-2.20.1-cp311-cp311-manylinux_2_17_armv7l.manylinux2014_armv7l.whl", hash = "sha256:25e9185e2d06c16ee438ed39bf62935ec436474a6ac4f9358524220f1b236e43", size = 1774694 },
    { url = "https://files.pythonhosted.org/packages/46/a4/08f12b5512f095963550a7cb49ae010e3f8f3f22b45e508c2cb4d7744fce/pydantic_core-2.20.1-cp311-cp311-manylinux_2_17_ppc64le.manylinux2014_ppc64le.whl", hash = "sha256:150906b40ff188a3260cbee25380e7494ee85048584998c1e66df0c7a11c17a6", size = 1976369 },
    { url = "https://files.pythonhosted.org/packages/15/59/b2495be4410462aedb399071c71884042a2c6443319cbf62d00b4a7ed7a5/pydantic_core-2.20.1-cp311-cp311-manylinux_2_17_s390x.manylinux2014_s390x.whl", hash = "sha256:8ad4aeb3e9a97286573c03df758fc7627aecdd02f1da04516a86dc159bf70121", size = 2691250 },
    { url = "https://files.pythonhosted.org/packages/3c/ae/fc99ce1ba791c9e9d1dee04ce80eef1dae5b25b27e3fc8e19f4e3f1348bf/pydantic_core-2.20.1-cp311-cp311-manylinux_2_17_x86_64.manylinux2014_x86_64.whl", hash = "sha256:d3f3ed29cd9f978c604708511a1f9c2fdcb6c38b9aae36a51905b8811ee5cbf1", size = 2061462 },
    { url = "https://files.pythonhosted.org/packages/44/bb/eb07cbe47cfd638603ce3cb8c220f1a054b821e666509e535f27ba07ca5f/pydantic_core-2.20.1-cp311-cp311-manylinux_2_5_i686.manylinux1_i686.whl", hash = "sha256:b0dae11d8f5ded51699c74d9548dcc5938e0804cc8298ec0aa0da95c21fff57b", size = 1893923 },
    { url = "https://files.pythonhosted.org/packages/ce/ef/5a52400553b8faa0e7f11fd7a2ba11e8d2feb50b540f9e7973c49b97eac0/pydantic_core-2.20.1-cp311-cp311-musllinux_1_1_aarch64.whl", hash = "sha256:faa6b09ee09433b87992fb5a2859efd1c264ddc37280d2dd5db502126d0e7f27", size = 1966779 },
    { url = "https://files.pythonhosted.org/packages/4c/5b/fb37fe341344d9651f5c5f579639cd97d50a457dc53901aa8f7e9f28beb9/pydantic_core-2.20.1-cp311-cp311-musllinux_1_1_x86_64.whl", hash = "sha256:9dc1b507c12eb0481d071f3c1808f0529ad41dc415d0ca11f7ebfc666e66a18b", size = 2109044 },
    { url = "https://files.pythonhosted.org/packages/70/1a/6f7278802dbc66716661618807ab0dfa4fc32b09d1235923bbbe8b3a5757/pydantic_core-2.20.1-cp311-none-win32.whl", hash = "sha256:fa2fddcb7107e0d1808086ca306dcade7df60a13a6c347a7acf1ec139aa6789a", size = 1708265 },
    { url = "https://files.pythonhosted.org/packages/35/7f/58758c42c61b0bdd585158586fecea295523d49933cb33664ea888162daf/pydantic_core-2.20.1-cp311-none-win_amd64.whl", hash = "sha256:40a783fb7ee353c50bd3853e626f15677ea527ae556429453685ae32280c19c2", size = 1901750 },
    { url = "https://files.pythonhosted.org/packages/6f/47/ef0d60ae23c41aced42921728650460dc831a0adf604bfa66b76028cb4d0/pydantic_core-2.20.1-cp312-cp312-macosx_10_12_x86_64.whl", hash = "sha256:595ba5be69b35777474fa07f80fc260ea71255656191adb22a8c53aba4479231", size = 1839225 },
    { url = "https://files.pythonhosted.org/packages/6a/23/430f2878c9cd977a61bb39f71751d9310ec55cee36b3d5bf1752c6341fd0/pydantic_core-2.20.1-cp312-cp312-macosx_11_0_arm64.whl", hash = "sha256:a4f55095ad087474999ee28d3398bae183a66be4823f753cd7d67dd0153427c9", size = 1768604 },
    { url = "https://files.pythonhosted.org/packages/9e/2b/ec4e7225dee79e0dc80ccc3c35ab33cc2c4bbb8a1a7ecf060e5e453651ec/pydantic_core-2.20.1-cp312-cp312-manylinux_2_17_aarch64.manylinux2014_aarch64.whl", hash = "sha256:f9aa05d09ecf4c75157197f27cdc9cfaeb7c5f15021c6373932bf3e124af029f", size = 1789767 },
    { url = "https://files.pythonhosted.org/packages/64/b0/38b24a1fa6d2f96af3148362e10737ec073768cd44d3ec21dca3be40a519/pydantic_core-2.20.1-cp312-cp312-manylinux_2_17_armv7l.manylinux2014_armv7l.whl", hash = "sha256:e97fdf088d4b31ff4ba35db26d9cc472ac7ef4a2ff2badeabf8d727b3377fc52", size = 1772061 },
    { url = "https://files.pythonhosted.org/packages/5e/da/bb73274c42cb60decfa61e9eb0c9029da78b3b9af0a9de0309dbc8ff87b6/pydantic_core-2.20.1-cp312-cp312-manylinux_2_17_ppc64le.manylinux2014_ppc64le.whl", hash = "sha256:bc633a9fe1eb87e250b5c57d389cf28998e4292336926b0b6cdaee353f89a237", size = 1974573 },
    { url = "https://files.pythonhosted.org/packages/c8/65/41693110fb3552556180460daffdb8bbeefb87fc026fd9aa4b849374015c/pydantic_core-2.20.1-cp312-cp312-manylinux_2_17_s390x.manylinux2014_s390x.whl", hash = "sha256:d573faf8eb7e6b1cbbcb4f5b247c60ca8be39fe2c674495df0eb4318303137fe", size = 2625596 },
    { url = "https://files.pythonhosted.org/packages/09/b3/a5a54b47cccd1ab661ed5775235c5e06924753c2d4817737c5667bfa19a8/pydantic_core-2.20.1-cp312-cp312-manylinux_2_17_x86_64.manylinux2014_x86_64.whl", hash = "sha256:26dc97754b57d2fd00ac2b24dfa341abffc380b823211994c4efac7f13b9e90e", size = 2099064 },
    { url = "https://files.pythonhosted.org/packages/52/fa/443a7a6ea54beaba45ff3a59f3d3e6e3004b7460bcfb0be77bcf98719d3b/pydantic_core-2.20.1-cp312-cp312-manylinux_2_5_i686.manylinux1_i686.whl", hash = "sha256:33499e85e739a4b60c9dac710c20a08dc73cb3240c9a0e22325e671b27b70d24", size = 1900345 },
    { url = "https://files.pythonhosted.org/packages/8e/e6/9aca9ffae60f9cdf0183069de3e271889b628d0fb175913fcb3db5618fb1/pydantic_core-2.20.1-cp312-cp312-musllinux_1_1_aarch64.whl", hash = "sha256:bebb4d6715c814597f85297c332297c6ce81e29436125ca59d1159b07f423eb1", size = 1968252 },
    { url = "https://files.pythonhosted.org/packages/46/5e/6c716810ea20a6419188992973a73c2fb4eb99cd382368d0637ddb6d3c99/pydantic_core-2.20.1-cp312-cp312-musllinux_1_1_x86_64.whl", hash = "sha256:516d9227919612425c8ef1c9b869bbbee249bc91912c8aaffb66116c0b447ebd", size = 2119191 },
    { url = "https://files.pythonhosted.org/packages/06/fc/6123b00a9240fbb9ae0babad7a005d51103d9a5d39c957a986f5cdd0c271/pydantic_core-2.20.1-cp312-none-win32.whl", hash = "sha256:469f29f9093c9d834432034d33f5fe45699e664f12a13bf38c04967ce233d688", size = 1717788 },
    { url = "https://files.pythonhosted.org/packages/d5/36/e61ad5a46607a469e2786f398cd671ebafcd9fb17f09a2359985c7228df5/pydantic_core-2.20.1-cp312-none-win_amd64.whl", hash = "sha256:035ede2e16da7281041f0e626459bcae33ed998cca6a0a007a5ebb73414ac72d", size = 1898188 },
    { url = "https://files.pythonhosted.org/packages/49/75/40b0e98b658fdba02a693b3bacb4c875a28bba87796c7b13975976597d8c/pydantic_core-2.20.1-cp313-cp313-macosx_10_12_x86_64.whl", hash = "sha256:0827505a5c87e8aa285dc31e9ec7f4a17c81a813d45f70b1d9164e03a813a686", size = 1838688 },
    { url = "https://files.pythonhosted.org/packages/75/02/d8ba2d4a266591a6a623c68b331b96523d4b62ab82a951794e3ed8907390/pydantic_core-2.20.1-cp313-cp313-macosx_11_0_arm64.whl", hash = "sha256:19c0fa39fa154e7e0b7f82f88ef85faa2a4c23cc65aae2f5aea625e3c13c735a", size = 1768409 },
    { url = "https://files.pythonhosted.org/packages/91/ae/25ecd9bc4ce4993e99a1a3c9ab111c082630c914260e129572fafed4ecc2/pydantic_core-2.20.1-cp313-cp313-manylinux_2_17_aarch64.manylinux2014_aarch64.whl", hash = "sha256:4aa223cd1e36b642092c326d694d8bf59b71ddddc94cdb752bbbb1c5c91d833b", size = 1789317 },
    { url = "https://files.pythonhosted.org/packages/7a/80/72057580681cdbe55699c367963d9c661b569a1d39338b4f6239faf36cdc/pydantic_core-2.20.1-cp313-cp313-manylinux_2_17_armv7l.manylinux2014_armv7l.whl", hash = "sha256:c336a6d235522a62fef872c6295a42ecb0c4e1d0f1a3e500fe949415761b8a19", size = 1771949 },
    { url = "https://files.pythonhosted.org/packages/a2/be/d9bbabc55b05019013180f141fcaf3b14dbe15ca7da550e95b60c321009a/pydantic_core-2.20.1-cp313-cp313-manylinux_2_17_ppc64le.manylinux2014_ppc64le.whl", hash = "sha256:7eb6a0587eded33aeefea9f916899d42b1799b7b14b8f8ff2753c0ac1741edac", size = 1974392 },
    { url = "https://files.pythonhosted.org/packages/79/2d/7bcd938c6afb0f40293283f5f09988b61fb0a4f1d180abe7c23a2f665f8e/pydantic_core-2.20.1-cp313-cp313-manylinux_2_17_s390x.manylinux2014_s390x.whl", hash = "sha256:70c8daf4faca8da5a6d655f9af86faf6ec2e1768f4b8b9d0226c02f3d6209703", size = 2625565 },
    { url = "https://files.pythonhosted.org/packages/ac/88/ca758e979457096008a4b16a064509028e3e092a1e85a5ed6c18ced8da88/pydantic_core-2.20.1-cp313-cp313-manylinux_2_17_x86_64.manylinux2014_x86_64.whl", hash = "sha256:e9fa4c9bf273ca41f940bceb86922a7667cd5bf90e95dbb157cbb8441008482c", size = 2098784 },
    { url = "https://files.pythonhosted.org/packages/eb/de/2fad6d63c3c42e472e985acb12ec45b7f56e42e6f4cd6dfbc5e87ee8678c/pydantic_core-2.20.1-cp313-cp313-manylinux_2_5_i686.manylinux1_i686.whl", hash = "sha256:11b71d67b4725e7e2a9f6e9c0ac1239bbc0c48cce3dc59f98635efc57d6dac83", size = 1900198 },
    { url = "https://files.pythonhosted.org/packages/fe/50/077c7f35b6488dc369a6d22993af3a37901e198630f38ac43391ca730f5b/pydantic_core-2.20.1-cp313-cp313-musllinux_1_1_aarch64.whl", hash = "sha256:270755f15174fb983890c49881e93f8f1b80f0b5e3a3cc1394a255706cabd203", size = 1968005 },
    { url = "https://files.pythonhosted.org/packages/5d/1f/f378631574ead46d636b9a04a80ff878b9365d4b361b1905ef1667d4182a/pydantic_core-2.20.1-cp313-cp313-musllinux_1_1_x86_64.whl", hash = "sha256:c81131869240e3e568916ef4c307f8b99583efaa60a8112ef27a366eefba8ef0", size = 2118920 },
    { url = "https://files.pythonhosted.org/packages/7a/ea/e4943f17df7a3031d709481fe4363d4624ae875a6409aec34c28c9e6cf59/pydantic_core-2.20.1-cp313-none-win32.whl", hash = "sha256:b91ced227c41aa29c672814f50dbb05ec93536abf8f43cd14ec9521ea09afe4e", size = 1717397 },
    { url = "https://files.pythonhosted.org/packages/13/63/b95781763e8d84207025071c0cec16d921c0163c7a9033ae4b9a0e020dc7/pydantic_core-2.20.1-cp313-none-win_amd64.whl", hash = "sha256:65db0f2eefcaad1a3950f498aabb4875c8890438bc80b19362cf633b87a8ab20", size = 1898013 },
]

[[package]]
name = "pygments"
version = "2.18.0"
source = { registry = "https://pypi.org/simple" }
sdist = { url = "https://files.pythonhosted.org/packages/8e/62/8336eff65bcbc8e4cb5d05b55faf041285951b6e80f33e2bff2024788f31/pygments-2.18.0.tar.gz", hash = "sha256:786ff802f32e91311bff3889f6e9a86e81505fe99f2735bb6d60ae0c5004f199", size = 4891905 }
wheels = [
    { url = "https://files.pythonhosted.org/packages/f7/3f/01c8b82017c199075f8f788d0d906b9ffbbc5a47dc9918a945e13d5a2bda/pygments-2.18.0-py3-none-any.whl", hash = "sha256:b8e6aca0523f3ab76fee51799c488e38782ac06eafcf95e7ba832985c8e7b13a", size = 1205513 },
]

[[package]]
name = "python-dotenv"
version = "1.0.1"
source = { registry = "https://pypi.org/simple" }
sdist = { url = "https://files.pythonhosted.org/packages/bc/57/e84d88dfe0aec03b7a2d4327012c1627ab5f03652216c63d49846d7a6c58/python-dotenv-1.0.1.tar.gz", hash = "sha256:e324ee90a023d808f1959c46bcbc04446a10ced277783dc6ee09987c37ec10ca", size = 39115 }
wheels = [
    { url = "https://files.pythonhosted.org/packages/6a/3e/b68c118422ec867fa7ab88444e1274aa40681c606d59ac27de5a5588f082/python_dotenv-1.0.1-py3-none-any.whl", hash = "sha256:f7b63ef50f1b690dddf550d03497b66d609393b40b564ed0d674909a68ebf16a", size = 19863 },
]

[[package]]
name = "python-multipart"
version = "0.0.12"
source = { registry = "https://pypi.org/simple" }
sdist = { url = "https://files.pythonhosted.org/packages/16/6e/7ecfe1366b9270f7f475c76fcfa28812493a6a1abd489b2433851a444f4f/python_multipart-0.0.12.tar.gz", hash = "sha256:045e1f98d719c1ce085ed7f7e1ef9d8ccc8c02ba02b5566d5f7521410ced58cb", size = 35713 }
wheels = [
    { url = "https://files.pythonhosted.org/packages/f5/0b/c316262244abea7481f95f1e91d7575f3dfcf6455d56d1ffe9839c582eb1/python_multipart-0.0.12-py3-none-any.whl", hash = "sha256:43dcf96cf65888a9cd3423544dd0d75ac10f7aa0c3c28a175bbcd00c9ce1aebf", size = 23246 },
]

[[package]]
name = "pyunormalize"
version = "16.0.0"
source = { registry = "https://pypi.org/simple" }
sdist = { url = "https://files.pythonhosted.org/packages/b3/08/568036c725dac746ecb267bb749ef930fb7907454fe69fce83c8557287fb/pyunormalize-16.0.0.tar.gz", hash = "sha256:2e1dfbb4a118154ae26f70710426a52a364b926c9191f764601f5a8cb12761f7", size = 49968 }
wheels = [
    { url = "https://files.pythonhosted.org/packages/39/f9/9d86e56f716e0651194a5ad58be9c146fcaf1de6901ac6f3cd3affeeb74e/pyunormalize-16.0.0-py3-none-any.whl", hash = "sha256:c647d95e5d1e2ea9a2f448d1d95d8518348df24eab5c3fd32d2b5c3300a49152", size = 49173 },
]

[[package]]
name = "pywin32"
version = "308"
source = { registry = "https://pypi.org/simple" }
wheels = [
    { url = "https://files.pythonhosted.org/packages/eb/e2/02652007469263fe1466e98439831d65d4ca80ea1a2df29abecedf7e47b7/pywin32-308-cp311-cp311-win32.whl", hash = "sha256:5d8c8015b24a7d6855b1550d8e660d8daa09983c80e5daf89a273e5c6fb5095a", size = 5928156 },
    { url = "https://files.pythonhosted.org/packages/48/ef/f4fb45e2196bc7ffe09cad0542d9aff66b0e33f6c0954b43e49c33cad7bd/pywin32-308-cp311-cp311-win_amd64.whl", hash = "sha256:575621b90f0dc2695fec346b2d6302faebd4f0f45c05ea29404cefe35d89442b", size = 6559559 },
    { url = "https://files.pythonhosted.org/packages/79/ef/68bb6aa865c5c9b11a35771329e95917b5559845bd75b65549407f9fc6b4/pywin32-308-cp311-cp311-win_arm64.whl", hash = "sha256:100a5442b7332070983c4cd03f2e906a5648a5104b8a7f50175f7906efd16bb6", size = 7972495 },
    { url = "https://files.pythonhosted.org/packages/00/7c/d00d6bdd96de4344e06c4afbf218bc86b54436a94c01c71a8701f613aa56/pywin32-308-cp312-cp312-win32.whl", hash = "sha256:587f3e19696f4bf96fde9d8a57cec74a57021ad5f204c9e627e15c33ff568897", size = 5939729 },
    { url = "https://files.pythonhosted.org/packages/21/27/0c8811fbc3ca188f93b5354e7c286eb91f80a53afa4e11007ef661afa746/pywin32-308-cp312-cp312-win_amd64.whl", hash = "sha256:00b3e11ef09ede56c6a43c71f2d31857cf7c54b0ab6e78ac659497abd2834f47", size = 6543015 },
    { url = "https://files.pythonhosted.org/packages/9d/0f/d40f8373608caed2255781a3ad9a51d03a594a1248cd632d6a298daca693/pywin32-308-cp312-cp312-win_arm64.whl", hash = "sha256:9b4de86c8d909aed15b7011182c8cab38c8850de36e6afb1f0db22b8959e3091", size = 7976033 },
    { url = "https://files.pythonhosted.org/packages/a9/a4/aa562d8935e3df5e49c161b427a3a2efad2ed4e9cf81c3de636f1fdddfd0/pywin32-308-cp313-cp313-win32.whl", hash = "sha256:1c44539a37a5b7b21d02ab34e6a4d314e0788f1690d65b48e9b0b89f31abbbed", size = 5938579 },
    { url = "https://files.pythonhosted.org/packages/c7/50/b0efb8bb66210da67a53ab95fd7a98826a97ee21f1d22949863e6d588b22/pywin32-308-cp313-cp313-win_amd64.whl", hash = "sha256:fd380990e792eaf6827fcb7e187b2b4b1cede0585e3d0c9e84201ec27b9905e4", size = 6542056 },
    { url = "https://files.pythonhosted.org/packages/26/df/2b63e3e4f2df0224f8aaf6d131f54fe4e8c96400eb9df563e2aae2e1a1f9/pywin32-308-cp313-cp313-win_arm64.whl", hash = "sha256:ef313c46d4c18dfb82a2431e3051ac8f112ccee1a34f29c263c583c568db63cd", size = 7974986 },
]

[[package]]
name = "pyyaml"
version = "6.0.2"
source = { registry = "https://pypi.org/simple" }
sdist = { url = "https://files.pythonhosted.org/packages/54/ed/79a089b6be93607fa5cdaedf301d7dfb23af5f25c398d5ead2525b063e17/pyyaml-6.0.2.tar.gz", hash = "sha256:d584d9ec91ad65861cc08d42e834324ef890a082e591037abe114850ff7bbc3e", size = 130631 }
wheels = [
    { url = "https://files.pythonhosted.org/packages/f8/aa/7af4e81f7acba21a4c6be026da38fd2b872ca46226673c89a758ebdc4fd2/PyYAML-6.0.2-cp311-cp311-macosx_10_9_x86_64.whl", hash = "sha256:cc1c1159b3d456576af7a3e4d1ba7e6924cb39de8f67111c735f6fc832082774", size = 184612 },
    { url = "https://files.pythonhosted.org/packages/8b/62/b9faa998fd185f65c1371643678e4d58254add437edb764a08c5a98fb986/PyYAML-6.0.2-cp311-cp311-macosx_11_0_arm64.whl", hash = "sha256:1e2120ef853f59c7419231f3bf4e7021f1b936f6ebd222406c3b60212205d2ee", size = 172040 },
    { url = "https://files.pythonhosted.org/packages/ad/0c/c804f5f922a9a6563bab712d8dcc70251e8af811fce4524d57c2c0fd49a4/PyYAML-6.0.2-cp311-cp311-manylinux_2_17_aarch64.manylinux2014_aarch64.whl", hash = "sha256:5d225db5a45f21e78dd9358e58a98702a0302f2659a3c6cd320564b75b86f47c", size = 736829 },
    { url = "https://files.pythonhosted.org/packages/51/16/6af8d6a6b210c8e54f1406a6b9481febf9c64a3109c541567e35a49aa2e7/PyYAML-6.0.2-cp311-cp311-manylinux_2_17_s390x.manylinux2014_s390x.whl", hash = "sha256:5ac9328ec4831237bec75defaf839f7d4564be1e6b25ac710bd1a96321cc8317", size = 764167 },
    { url = "https://files.pythonhosted.org/packages/75/e4/2c27590dfc9992f73aabbeb9241ae20220bd9452df27483b6e56d3975cc5/PyYAML-6.0.2-cp311-cp311-manylinux_2_17_x86_64.manylinux2014_x86_64.whl", hash = "sha256:3ad2a3decf9aaba3d29c8f537ac4b243e36bef957511b4766cb0057d32b0be85", size = 762952 },
    { url = "https://files.pythonhosted.org/packages/9b/97/ecc1abf4a823f5ac61941a9c00fe501b02ac3ab0e373c3857f7d4b83e2b6/PyYAML-6.0.2-cp311-cp311-musllinux_1_1_aarch64.whl", hash = "sha256:ff3824dc5261f50c9b0dfb3be22b4567a6f938ccce4587b38952d85fd9e9afe4", size = 735301 },
    { url = "https://files.pythonhosted.org/packages/45/73/0f49dacd6e82c9430e46f4a027baa4ca205e8b0a9dce1397f44edc23559d/PyYAML-6.0.2-cp311-cp311-musllinux_1_1_x86_64.whl", hash = "sha256:797b4f722ffa07cc8d62053e4cff1486fa6dc094105d13fea7b1de7d8bf71c9e", size = 756638 },
    { url = "https://files.pythonhosted.org/packages/22/5f/956f0f9fc65223a58fbc14459bf34b4cc48dec52e00535c79b8db361aabd/PyYAML-6.0.2-cp311-cp311-win32.whl", hash = "sha256:11d8f3dd2b9c1207dcaf2ee0bbbfd5991f571186ec9cc78427ba5bd32afae4b5", size = 143850 },
    { url = "https://files.pythonhosted.org/packages/ed/23/8da0bbe2ab9dcdd11f4f4557ccaf95c10b9811b13ecced089d43ce59c3c8/PyYAML-6.0.2-cp311-cp311-win_amd64.whl", hash = "sha256:e10ce637b18caea04431ce14fabcf5c64a1c61ec9c56b071a4b7ca131ca52d44", size = 161980 },
    { url = "https://files.pythonhosted.org/packages/86/0c/c581167fc46d6d6d7ddcfb8c843a4de25bdd27e4466938109ca68492292c/PyYAML-6.0.2-cp312-cp312-macosx_10_9_x86_64.whl", hash = "sha256:c70c95198c015b85feafc136515252a261a84561b7b1d51e3384e0655ddf25ab", size = 183873 },
    { url = "https://files.pythonhosted.org/packages/a8/0c/38374f5bb272c051e2a69281d71cba6fdb983413e6758b84482905e29a5d/PyYAML-6.0.2-cp312-cp312-macosx_11_0_arm64.whl", hash = "sha256:ce826d6ef20b1bc864f0a68340c8b3287705cae2f8b4b1d932177dcc76721725", size = 173302 },
    { url = "https://files.pythonhosted.org/packages/c3/93/9916574aa8c00aa06bbac729972eb1071d002b8e158bd0e83a3b9a20a1f7/PyYAML-6.0.2-cp312-cp312-manylinux_2_17_aarch64.manylinux2014_aarch64.whl", hash = "sha256:1f71ea527786de97d1a0cc0eacd1defc0985dcf6b3f17bb77dcfc8c34bec4dc5", size = 739154 },
    { url = "https://files.pythonhosted.org/packages/95/0f/b8938f1cbd09739c6da569d172531567dbcc9789e0029aa070856f123984/PyYAML-6.0.2-cp312-cp312-manylinux_2_17_s390x.manylinux2014_s390x.whl", hash = "sha256:9b22676e8097e9e22e36d6b7bda33190d0d400f345f23d4065d48f4ca7ae0425", size = 766223 },
    { url = "https://files.pythonhosted.org/packages/b9/2b/614b4752f2e127db5cc206abc23a8c19678e92b23c3db30fc86ab731d3bd/PyYAML-6.0.2-cp312-cp312-manylinux_2_17_x86_64.manylinux2014_x86_64.whl", hash = "sha256:80bab7bfc629882493af4aa31a4cfa43a4c57c83813253626916b8c7ada83476", size = 767542 },
    { url = "https://files.pythonhosted.org/packages/d4/00/dd137d5bcc7efea1836d6264f049359861cf548469d18da90cd8216cf05f/PyYAML-6.0.2-cp312-cp312-musllinux_1_1_aarch64.whl", hash = "sha256:0833f8694549e586547b576dcfaba4a6b55b9e96098b36cdc7ebefe667dfed48", size = 731164 },
    { url = "https://files.pythonhosted.org/packages/c9/1f/4f998c900485e5c0ef43838363ba4a9723ac0ad73a9dc42068b12aaba4e4/PyYAML-6.0.2-cp312-cp312-musllinux_1_1_x86_64.whl", hash = "sha256:8b9c7197f7cb2738065c481a0461e50ad02f18c78cd75775628afb4d7137fb3b", size = 756611 },
    { url = "https://files.pythonhosted.org/packages/df/d1/f5a275fdb252768b7a11ec63585bc38d0e87c9e05668a139fea92b80634c/PyYAML-6.0.2-cp312-cp312-win32.whl", hash = "sha256:ef6107725bd54b262d6dedcc2af448a266975032bc85ef0172c5f059da6325b4", size = 140591 },
    { url = "https://files.pythonhosted.org/packages/0c/e8/4f648c598b17c3d06e8753d7d13d57542b30d56e6c2dedf9c331ae56312e/PyYAML-6.0.2-cp312-cp312-win_amd64.whl", hash = "sha256:7e7401d0de89a9a855c839bc697c079a4af81cf878373abd7dc625847d25cbd8", size = 156338 },
    { url = "https://files.pythonhosted.org/packages/ef/e3/3af305b830494fa85d95f6d95ef7fa73f2ee1cc8ef5b495c7c3269fb835f/PyYAML-6.0.2-cp313-cp313-macosx_10_13_x86_64.whl", hash = "sha256:efdca5630322a10774e8e98e1af481aad470dd62c3170801852d752aa7a783ba", size = 181309 },
    { url = "https://files.pythonhosted.org/packages/45/9f/3b1c20a0b7a3200524eb0076cc027a970d320bd3a6592873c85c92a08731/PyYAML-6.0.2-cp313-cp313-macosx_11_0_arm64.whl", hash = "sha256:50187695423ffe49e2deacb8cd10510bc361faac997de9efef88badc3bb9e2d1", size = 171679 },
    { url = "https://files.pythonhosted.org/packages/7c/9a/337322f27005c33bcb656c655fa78325b730324c78620e8328ae28b64d0c/PyYAML-6.0.2-cp313-cp313-manylinux_2_17_aarch64.manylinux2014_aarch64.whl", hash = "sha256:0ffe8360bab4910ef1b9e87fb812d8bc0a308b0d0eef8c8f44e0254ab3b07133", size = 733428 },
    { url = "https://files.pythonhosted.org/packages/a3/69/864fbe19e6c18ea3cc196cbe5d392175b4cf3d5d0ac1403ec3f2d237ebb5/PyYAML-6.0.2-cp313-cp313-manylinux_2_17_s390x.manylinux2014_s390x.whl", hash = "sha256:17e311b6c678207928d649faa7cb0d7b4c26a0ba73d41e99c4fff6b6c3276484", size = 763361 },
    { url = "https://files.pythonhosted.org/packages/04/24/b7721e4845c2f162d26f50521b825fb061bc0a5afcf9a386840f23ea19fa/PyYAML-6.0.2-cp313-cp313-manylinux_2_17_x86_64.manylinux2014_x86_64.whl", hash = "sha256:70b189594dbe54f75ab3a1acec5f1e3faa7e8cf2f1e08d9b561cb41b845f69d5", size = 759523 },
    { url = "https://files.pythonhosted.org/packages/2b/b2/e3234f59ba06559c6ff63c4e10baea10e5e7df868092bf9ab40e5b9c56b6/PyYAML-6.0.2-cp313-cp313-musllinux_1_1_aarch64.whl", hash = "sha256:41e4e3953a79407c794916fa277a82531dd93aad34e29c2a514c2c0c5fe971cc", size = 726660 },
    { url = "https://files.pythonhosted.org/packages/fe/0f/25911a9f080464c59fab9027482f822b86bf0608957a5fcc6eaac85aa515/PyYAML-6.0.2-cp313-cp313-musllinux_1_1_x86_64.whl", hash = "sha256:68ccc6023a3400877818152ad9a1033e3db8625d899c72eacb5a668902e4d652", size = 751597 },
    { url = "https://files.pythonhosted.org/packages/14/0d/e2c3b43bbce3cf6bd97c840b46088a3031085179e596d4929729d8d68270/PyYAML-6.0.2-cp313-cp313-win32.whl", hash = "sha256:bc2fa7c6b47d6bc618dd7fb02ef6fdedb1090ec036abab80d4681424b84c1183", size = 140527 },
    { url = "https://files.pythonhosted.org/packages/fa/de/02b54f42487e3d3c6efb3f89428677074ca7bf43aae402517bc7cca949f3/PyYAML-6.0.2-cp313-cp313-win_amd64.whl", hash = "sha256:8388ee1976c416731879ac16da0aff3f63b286ffdd57cdeb95f3f2e085687563", size = 156446 },
]

[[package]]
name = "regex"
version = "2024.9.11"
source = { registry = "https://pypi.org/simple" }
sdist = { url = "https://files.pythonhosted.org/packages/f9/38/148df33b4dbca3bd069b963acab5e0fa1a9dbd6820f8c322d0dd6faeff96/regex-2024.9.11.tar.gz", hash = "sha256:6c188c307e8433bcb63dc1915022deb553b4203a70722fc542c363bf120a01fd", size = 399403 }
wheels = [
    { url = "https://files.pythonhosted.org/packages/86/a1/d526b7b6095a0019aa360948c143aacfeb029919c898701ce7763bbe4c15/regex-2024.9.11-cp311-cp311-macosx_10_9_universal2.whl", hash = "sha256:2cce2449e5927a0bf084d346da6cd5eb016b2beca10d0013ab50e3c226ffc0df", size = 482483 },
    { url = "https://files.pythonhosted.org/packages/32/d9/bfdd153179867c275719e381e1e8e84a97bd186740456a0dcb3e7125c205/regex-2024.9.11-cp311-cp311-macosx_10_9_x86_64.whl", hash = "sha256:3b37fa423beefa44919e009745ccbf353d8c981516e807995b2bd11c2c77d268", size = 287442 },
    { url = "https://files.pythonhosted.org/packages/33/c4/60f3370735135e3a8d673ddcdb2507a8560d0e759e1398d366e43d000253/regex-2024.9.11-cp311-cp311-macosx_11_0_arm64.whl", hash = "sha256:64ce2799bd75039b480cc0360907c4fb2f50022f030bf9e7a8705b636e408fad", size = 284561 },
    { url = "https://files.pythonhosted.org/packages/b1/51/91a5ebdff17f9ec4973cb0aa9d37635efec1c6868654bbc25d1543aca4ec/regex-2024.9.11-cp311-cp311-manylinux_2_17_aarch64.manylinux2014_aarch64.whl", hash = "sha256:a4cc92bb6db56ab0c1cbd17294e14f5e9224f0cc6521167ef388332604e92679", size = 791779 },
    { url = "https://files.pythonhosted.org/packages/07/4a/022c5e6f0891a90cd7eb3d664d6c58ce2aba48bff107b00013f3d6167069/regex-2024.9.11-cp311-cp311-manylinux_2_17_ppc64le.manylinux2014_ppc64le.whl", hash = "sha256:d05ac6fa06959c4172eccd99a222e1fbf17b5670c4d596cb1e5cde99600674c4", size = 832605 },
    { url = "https://files.pythonhosted.org/packages/ac/1c/3793990c8c83ca04e018151ddda83b83ecc41d89964f0f17749f027fc44d/regex-2024.9.11-cp311-cp311-manylinux_2_17_s390x.manylinux2014_s390x.whl", hash = "sha256:040562757795eeea356394a7fb13076ad4f99d3c62ab0f8bdfb21f99a1f85664", size = 818556 },
    { url = "https://files.pythonhosted.org/packages/e9/5c/8b385afbfacb853730682c57be56225f9fe275c5bf02ac1fc88edbff316d/regex-2024.9.11-cp311-cp311-manylinux_2_17_x86_64.manylinux2014_x86_64.whl", hash = "sha256:6113c008a7780792efc80f9dfe10ba0cd043cbf8dc9a76ef757850f51b4edc50", size = 792808 },
    { url = "https://files.pythonhosted.org/packages/9b/8b/a4723a838b53c771e9240951adde6af58c829fb6a6a28f554e8131f53839/regex-2024.9.11-cp311-cp311-manylinux_2_5_i686.manylinux1_i686.manylinux_2_17_i686.manylinux2014_i686.whl", hash = "sha256:8e5fb5f77c8745a60105403a774fe2c1759b71d3e7b4ca237a5e67ad066c7199", size = 781115 },
    { url = "https://files.pythonhosted.org/packages/83/5f/031a04b6017033d65b261259c09043c06f4ef2d4eac841d0649d76d69541/regex-2024.9.11-cp311-cp311-musllinux_1_2_aarch64.whl", hash = "sha256:54d9ff35d4515debf14bc27f1e3b38bfc453eff3220f5bce159642fa762fe5d4", size = 778155 },
    { url = "https://files.pythonhosted.org/packages/fd/cd/4660756070b03ce4a66663a43f6c6e7ebc2266cc6b4c586c167917185eb4/regex-2024.9.11-cp311-cp311-musllinux_1_2_i686.whl", hash = "sha256:df5cbb1fbc74a8305b6065d4ade43b993be03dbe0f8b30032cced0d7740994bd", size = 784614 },
    { url = "https://files.pythonhosted.org/packages/93/8d/65b9bea7df120a7be8337c415b6d256ba786cbc9107cebba3bf8ff09da99/regex-2024.9.11-cp311-cp311-musllinux_1_2_ppc64le.whl", hash = "sha256:7fb89ee5d106e4a7a51bce305ac4efb981536301895f7bdcf93ec92ae0d91c7f", size = 853744 },
    { url = "https://files.pythonhosted.org/packages/96/a7/fba1eae75eb53a704475baf11bd44b3e6ccb95b316955027eb7748f24ef8/regex-2024.9.11-cp311-cp311-musllinux_1_2_s390x.whl", hash = "sha256:a738b937d512b30bf75995c0159c0ddf9eec0775c9d72ac0202076c72f24aa96", size = 855890 },
    { url = "https://files.pythonhosted.org/packages/45/14/d864b2db80a1a3358534392373e8a281d95b28c29c87d8548aed58813910/regex-2024.9.11-cp311-cp311-musllinux_1_2_x86_64.whl", hash = "sha256:e28f9faeb14b6f23ac55bfbbfd3643f5c7c18ede093977f1df249f73fd22c7b1", size = 781887 },
    { url = "https://files.pythonhosted.org/packages/4d/a9/bfb29b3de3eb11dc9b412603437023b8e6c02fb4e11311863d9bf62c403a/regex-2024.9.11-cp311-cp311-win32.whl", hash = "sha256:18e707ce6c92d7282dfce370cd205098384b8ee21544e7cb29b8aab955b66fa9", size = 261644 },
    { url = "https://files.pythonhosted.org/packages/c7/ab/1ad2511cf6a208fde57fafe49829cab8ca018128ab0d0b48973d8218634a/regex-2024.9.11-cp311-cp311-win_amd64.whl", hash = "sha256:313ea15e5ff2a8cbbad96ccef6be638393041b0a7863183c2d31e0c6116688cf", size = 274033 },
    { url = "https://files.pythonhosted.org/packages/6e/92/407531450762bed778eedbde04407f68cbd75d13cee96c6f8d6903d9c6c1/regex-2024.9.11-cp312-cp312-macosx_10_9_universal2.whl", hash = "sha256:b0d0a6c64fcc4ef9c69bd5b3b3626cc3776520a1637d8abaa62b9edc147a58f7", size = 483590 },
    { url = "https://files.pythonhosted.org/packages/8e/a2/048acbc5ae1f615adc6cba36cc45734e679b5f1e4e58c3c77f0ed611d4e2/regex-2024.9.11-cp312-cp312-macosx_10_9_x86_64.whl", hash = "sha256:49b0e06786ea663f933f3710a51e9385ce0cba0ea56b67107fd841a55d56a231", size = 288175 },
    { url = "https://files.pythonhosted.org/packages/8a/ea/909d8620329ab710dfaf7b4adee41242ab7c9b95ea8d838e9bfe76244259/regex-2024.9.11-cp312-cp312-macosx_11_0_arm64.whl", hash = "sha256:5b513b6997a0b2f10e4fd3a1313568e373926e8c252bd76c960f96fd039cd28d", size = 284749 },
    { url = "https://files.pythonhosted.org/packages/ca/fa/521eb683b916389b4975337873e66954e0f6d8f91bd5774164a57b503185/regex-2024.9.11-cp312-cp312-manylinux_2_17_aarch64.manylinux2014_aarch64.whl", hash = "sha256:ee439691d8c23e76f9802c42a95cfeebf9d47cf4ffd06f18489122dbb0a7ad64", size = 795181 },
    { url = "https://files.pythonhosted.org/packages/28/db/63047feddc3280cc242f9c74f7aeddc6ee662b1835f00046f57d5630c827/regex-2024.9.11-cp312-cp312-manylinux_2_17_ppc64le.manylinux2014_ppc64le.whl", hash = "sha256:a8f877c89719d759e52783f7fe6e1c67121076b87b40542966c02de5503ace42", size = 835842 },
    { url = "https://files.pythonhosted.org/packages/e3/94/86adc259ff8ec26edf35fcca7e334566c1805c7493b192cb09679f9c3dee/regex-2024.9.11-cp312-cp312-manylinux_2_17_s390x.manylinux2014_s390x.whl", hash = "sha256:23b30c62d0f16827f2ae9f2bb87619bc4fba2044911e2e6c2eb1af0161cdb766", size = 823533 },
    { url = "https://files.pythonhosted.org/packages/29/52/84662b6636061277cb857f658518aa7db6672bc6d1a3f503ccd5aefc581e/regex-2024.9.11-cp312-cp312-manylinux_2_17_x86_64.manylinux2014_x86_64.whl", hash = "sha256:85ab7824093d8f10d44330fe1e6493f756f252d145323dd17ab6b48733ff6c0a", size = 797037 },
    { url = "https://files.pythonhosted.org/packages/c3/2a/cd4675dd987e4a7505f0364a958bc41f3b84942de9efaad0ef9a2646681c/regex-2024.9.11-cp312-cp312-manylinux_2_5_i686.manylinux1_i686.manylinux_2_17_i686.manylinux2014_i686.whl", hash = "sha256:8dee5b4810a89447151999428fe096977346cf2f29f4d5e29609d2e19e0199c9", size = 784106 },
    { url = "https://files.pythonhosted.org/packages/6f/75/3ea7ec29de0bbf42f21f812f48781d41e627d57a634f3f23947c9a46e303/regex-2024.9.11-cp312-cp312-musllinux_1_2_aarch64.whl", hash = "sha256:98eeee2f2e63edae2181c886d7911ce502e1292794f4c5ee71e60e23e8d26b5d", size = 782468 },
    { url = "https://files.pythonhosted.org/packages/d3/67/15519d69b52c252b270e679cb578e22e0c02b8dd4e361f2b04efcc7f2335/regex-2024.9.11-cp312-cp312-musllinux_1_2_i686.whl", hash = "sha256:57fdd2e0b2694ce6fc2e5ccf189789c3e2962916fb38779d3e3521ff8fe7a822", size = 790324 },
    { url = "https://files.pythonhosted.org/packages/9c/71/eff77d3fe7ba08ab0672920059ec30d63fa7e41aa0fb61c562726e9bd721/regex-2024.9.11-cp312-cp312-musllinux_1_2_ppc64le.whl", hash = "sha256:d552c78411f60b1fdaafd117a1fca2f02e562e309223b9d44b7de8be451ec5e0", size = 860214 },
    { url = "https://files.pythonhosted.org/packages/81/11/e1bdf84a72372e56f1ea4b833dd583b822a23138a616ace7ab57a0e11556/regex-2024.9.11-cp312-cp312-musllinux_1_2_s390x.whl", hash = "sha256:a0b2b80321c2ed3fcf0385ec9e51a12253c50f146fddb2abbb10f033fe3d049a", size = 859420 },
    { url = "https://files.pythonhosted.org/packages/ea/75/9753e9dcebfa7c3645563ef5c8a58f3a47e799c872165f37c55737dadd3e/regex-2024.9.11-cp312-cp312-musllinux_1_2_x86_64.whl", hash = "sha256:18406efb2f5a0e57e3a5881cd9354c1512d3bb4f5c45d96d110a66114d84d23a", size = 787333 },
    { url = "https://files.pythonhosted.org/packages/bc/4e/ba1cbca93141f7416624b3ae63573e785d4bc1834c8be44a8f0747919eca/regex-2024.9.11-cp312-cp312-win32.whl", hash = "sha256:e464b467f1588e2c42d26814231edecbcfe77f5ac414d92cbf4e7b55b2c2a776", size = 262058 },
    { url = "https://files.pythonhosted.org/packages/6e/16/efc5f194778bf43e5888209e5cec4b258005d37c613b67ae137df3b89c53/regex-2024.9.11-cp312-cp312-win_amd64.whl", hash = "sha256:9e8719792ca63c6b8340380352c24dcb8cd7ec49dae36e963742a275dfae6009", size = 273526 },
    { url = "https://files.pythonhosted.org/packages/93/0a/d1c6b9af1ff1e36832fe38d74d5c5bab913f2bdcbbd6bc0e7f3ce8b2f577/regex-2024.9.11-cp313-cp313-macosx_10_13_universal2.whl", hash = "sha256:c157bb447303070f256e084668b702073db99bbb61d44f85d811025fcf38f784", size = 483376 },
    { url = "https://files.pythonhosted.org/packages/a4/42/5910a050c105d7f750a72dcb49c30220c3ae4e2654e54aaaa0e9bc0584cb/regex-2024.9.11-cp313-cp313-macosx_10_13_x86_64.whl", hash = "sha256:4db21ece84dfeefc5d8a3863f101995de646c6cb0536952c321a2650aa202c36", size = 288112 },
    { url = "https://files.pythonhosted.org/packages/8d/56/0c262aff0e9224fa7ffce47b5458d373f4d3e3ff84e99b5ff0cb15e0b5b2/regex-2024.9.11-cp313-cp313-macosx_11_0_arm64.whl", hash = "sha256:220e92a30b426daf23bb67a7962900ed4613589bab80382be09b48896d211e92", size = 284608 },
    { url = "https://files.pythonhosted.org/packages/b9/54/9fe8f9aec5007bbbbce28ba3d2e3eaca425f95387b7d1e84f0d137d25237/regex-2024.9.11-cp313-cp313-manylinux_2_17_aarch64.manylinux2014_aarch64.whl", hash = "sha256:eb1ae19e64c14c7ec1995f40bd932448713d3c73509e82d8cd7744dc00e29e86", size = 795337 },
    { url = "https://files.pythonhosted.org/packages/b2/e7/6b2f642c3cded271c4f16cc4daa7231be544d30fe2b168e0223724b49a61/regex-2024.9.11-cp313-cp313-manylinux_2_17_ppc64le.manylinux2014_ppc64le.whl", hash = "sha256:f47cd43a5bfa48f86925fe26fbdd0a488ff15b62468abb5d2a1e092a4fb10e85", size = 835848 },
    { url = "https://files.pythonhosted.org/packages/cd/9e/187363bdf5d8c0e4662117b92aa32bf52f8f09620ae93abc7537d96d3311/regex-2024.9.11-cp313-cp313-manylinux_2_17_s390x.manylinux2014_s390x.whl", hash = "sha256:9d4a76b96f398697fe01117093613166e6aa8195d63f1b4ec3f21ab637632963", size = 823503 },
    { url = "https://files.pythonhosted.org/packages/f8/10/601303b8ee93589f879664b0cfd3127949ff32b17f9b6c490fb201106c4d/regex-2024.9.11-cp313-cp313-manylinux_2_17_x86_64.manylinux2014_x86_64.whl", hash = "sha256:0ea51dcc0835eea2ea31d66456210a4e01a076d820e9039b04ae8d17ac11dee6", size = 797049 },
    { url = "https://files.pythonhosted.org/packages/ef/1c/ea200f61ce9f341763f2717ab4daebe4422d83e9fd4ac5e33435fd3a148d/regex-2024.9.11-cp313-cp313-manylinux_2_5_i686.manylinux1_i686.manylinux_2_17_i686.manylinux2014_i686.whl", hash = "sha256:b7aaa315101c6567a9a45d2839322c51c8d6e81f67683d529512f5bcfb99c802", size = 784144 },
    { url = "https://files.pythonhosted.org/packages/d8/5c/d2429be49ef3292def7688401d3deb11702c13dcaecdc71d2b407421275b/regex-2024.9.11-cp313-cp313-musllinux_1_2_aarch64.whl", hash = "sha256:c57d08ad67aba97af57a7263c2d9006d5c404d721c5f7542f077f109ec2a4a29", size = 782483 },
    { url = "https://files.pythonhosted.org/packages/12/d9/cbc30f2ff7164f3b26a7760f87c54bf8b2faed286f60efd80350a51c5b99/regex-2024.9.11-cp313-cp313-musllinux_1_2_i686.whl", hash = "sha256:f8404bf61298bb6f8224bb9176c1424548ee1181130818fcd2cbffddc768bed8", size = 790320 },
    { url = "https://files.pythonhosted.org/packages/19/1d/43ed03a236313639da5a45e61bc553c8d41e925bcf29b0f8ecff0c2c3f25/regex-2024.9.11-cp313-cp313-musllinux_1_2_ppc64le.whl", hash = "sha256:dd4490a33eb909ef5078ab20f5f000087afa2a4daa27b4c072ccb3cb3050ad84", size = 860435 },
    { url = "https://files.pythonhosted.org/packages/34/4f/5d04da61c7c56e785058a46349f7285ae3ebc0726c6ea7c5c70600a52233/regex-2024.9.11-cp313-cp313-musllinux_1_2_s390x.whl", hash = "sha256:eee9130eaad130649fd73e5cd92f60e55708952260ede70da64de420cdcad554", size = 859571 },
    { url = "https://files.pythonhosted.org/packages/12/7f/8398c8155a3c70703a8e91c29532558186558e1aea44144b382faa2a6f7a/regex-2024.9.11-cp313-cp313-musllinux_1_2_x86_64.whl", hash = "sha256:6a2644a93da36c784e546de579ec1806bfd2763ef47babc1b03d765fe560c9f8", size = 787398 },
    { url = "https://files.pythonhosted.org/packages/58/3a/f5903977647a9a7e46d5535e9e96c194304aeeca7501240509bde2f9e17f/regex-2024.9.11-cp313-cp313-win32.whl", hash = "sha256:e997fd30430c57138adc06bba4c7c2968fb13d101e57dd5bb9355bf8ce3fa7e8", size = 262035 },
    { url = "https://files.pythonhosted.org/packages/ff/80/51ba3a4b7482f6011095b3a036e07374f64de180b7d870b704ed22509002/regex-2024.9.11-cp313-cp313-win_amd64.whl", hash = "sha256:042c55879cfeb21a8adacc84ea347721d3d83a159da6acdf1116859e2427c43f", size = 273510 },
]

[[package]]
name = "requests"
version = "2.31.0"
source = { registry = "https://pypi.org/simple" }
dependencies = [
    { name = "certifi" },
    { name = "charset-normalizer" },
    { name = "idna" },
    { name = "urllib3" },
]
sdist = { url = "https://files.pythonhosted.org/packages/9d/be/10918a2eac4ae9f02f6cfe6414b7a155ccd8f7f9d4380d62fd5b955065c3/requests-2.31.0.tar.gz", hash = "sha256:942c5a758f98d790eaed1a29cb6eefc7ffb0d1cf7af05c3d2791656dbd6ad1e1", size = 110794 }
wheels = [
    { url = "https://files.pythonhosted.org/packages/70/8e/0e2d847013cb52cd35b38c009bb167a1a26b2ce6cd6965bf26b47bc0bf44/requests-2.31.0-py3-none-any.whl", hash = "sha256:58cd2187c01e70e6e26505bca751777aa9f2ee0b7f4300988b709f44e013003f", size = 62574 },
]

[[package]]
name = "requests-oauthlib"
version = "1.3.1"
source = { registry = "https://pypi.org/simple" }
dependencies = [
    { name = "oauthlib" },
    { name = "requests" },
]
sdist = { url = "https://files.pythonhosted.org/packages/95/52/531ef197b426646f26b53815a7d2a67cb7a331ef098bb276db26a68ac49f/requests-oauthlib-1.3.1.tar.gz", hash = "sha256:75beac4a47881eeb94d5ea5d6ad31ef88856affe2332b9aafb52c6452ccf0d7a", size = 52027 }
wheels = [
    { url = "https://files.pythonhosted.org/packages/6f/bb/5deac77a9af870143c684ab46a7934038a53eb4aa975bc0687ed6ca2c610/requests_oauthlib-1.3.1-py2.py3-none-any.whl", hash = "sha256:2577c501a2fb8d05a304c09d090d6e47c306fef15809d102b327cf8364bddab5", size = 23892 },
]

[[package]]
name = "rich"
version = "13.9.3"
source = { registry = "https://pypi.org/simple" }
dependencies = [
    { name = "markdown-it-py" },
    { name = "pygments" },
]
sdist = { url = "https://files.pythonhosted.org/packages/d9/e9/cf9ef5245d835065e6673781dbd4b8911d352fb770d56cf0879cf11b7ee1/rich-13.9.3.tar.gz", hash = "sha256:bc1e01b899537598cf02579d2b9f4a415104d3fc439313a7a2c165d76557a08e", size = 222889 }
wheels = [
    { url = "https://files.pythonhosted.org/packages/9a/e2/10e9819cf4a20bd8ea2f5dabafc2e6bf4a78d6a0965daeb60a4b34d1c11f/rich-13.9.3-py3-none-any.whl", hash = "sha256:9836f5096eb2172c9e77df411c1b009bace4193d6a481d534fea75ebba758283", size = 242157 },
]

[[package]]
name = "rlp"
version = "4.0.1"
source = { registry = "https://pypi.org/simple" }
dependencies = [
    { name = "eth-utils" },
]
sdist = { url = "https://files.pythonhosted.org/packages/0f/49/bcd4d3f9210ed78749eab04d236eeb98f98fbcc16977f308ee4637c1bad8/rlp-4.0.1.tar.gz", hash = "sha256:bcefb11013dfadf8902642337923bd0c786dc8a27cb4c21da6e154e52869ecb1", size = 33710 }
wheels = [
    { url = "https://files.pythonhosted.org/packages/59/03/3ae09a1c43657d17530dd98de6e381cc66ac514daa67000ccf99afc808fc/rlp-4.0.1-py3-none-any.whl", hash = "sha256:ff6846c3c27b97ee0492373aa074a7c3046aadd973320f4fffa7ac45564b0258", size = 20639 },
]

[[package]]
name = "shellingham"
version = "1.5.4"
source = { registry = "https://pypi.org/simple" }
sdist = { url = "https://files.pythonhosted.org/packages/58/15/8b3609fd3830ef7b27b655beb4b4e9c62313a4e8da8c676e142cc210d58e/shellingham-1.5.4.tar.gz", hash = "sha256:8dbca0739d487e5bd35ab3ca4b36e11c4078f3a234bfce294b0a0291363404de", size = 10310 }
wheels = [
    { url = "https://files.pythonhosted.org/packages/e0/f9/0595336914c5619e5f28a1fb793285925a8cd4b432c9da0a987836c7f822/shellingham-1.5.4-py2.py3-none-any.whl", hash = "sha256:7ecfff8f2fd72616f7481040475a65b2bf8af90a56c89140852d1120324e8686", size = 9755 },
]

[[package]]
name = "sniffio"
version = "1.3.1"
source = { registry = "https://pypi.org/simple" }
sdist = { url = "https://files.pythonhosted.org/packages/a2/87/a6771e1546d97e7e041b6ae58d80074f81b7d5121207425c964ddf5cfdbd/sniffio-1.3.1.tar.gz", hash = "sha256:f4324edc670a0f49750a81b895f35c3adb843cca46f0530f79fc1babb23789dc", size = 20372 }
wheels = [
    { url = "https://files.pythonhosted.org/packages/e9/44/75a9c9421471a6c4805dbf2356f7c181a29c1879239abab1ea2cc8f38b40/sniffio-1.3.1-py3-none-any.whl", hash = "sha256:2f6da418d1f1e0fddd844478f41680e794e6051915791a034ff65e5f100525a2", size = 10235 },
]

[[package]]
name = "sqlalchemy"
version = "2.0.31"
source = { registry = "https://pypi.org/simple" }
dependencies = [
    { name = "greenlet", marker = "(python_full_version < '3.13' and platform_machine == 'AMD64') or (python_full_version < '3.13' and platform_machine == 'WIN32') or (python_full_version < '3.13' and platform_machine == 'aarch64') or (python_full_version < '3.13' and platform_machine == 'amd64') or (python_full_version < '3.13' and platform_machine == 'ppc64le') or (python_full_version < '3.13' and platform_machine == 'win32') or (python_full_version < '3.13' and platform_machine == 'x86_64')" },
    { name = "typing-extensions" },
]
sdist = { url = "https://files.pythonhosted.org/packages/ba/7d/e3312ae374fe7a4af7e1494735125a714a0907317c829ab8d8a31d89ded4/SQLAlchemy-2.0.31.tar.gz", hash = "sha256:b607489dd4a54de56984a0c7656247504bd5523d9d0ba799aef59d4add009484", size = 9524110 }
wheels = [
    { url = "https://files.pythonhosted.org/packages/54/11/aa208123e8cc01bef67e6af9942ba94c1009687def8474eb0eff74ea6cf1/SQLAlchemy-2.0.31-cp311-cp311-macosx_10_9_x86_64.whl", hash = "sha256:f68470edd70c3ac3b6cd5c2a22a8daf18415203ca1b036aaeb9b0fb6f54e8298", size = 2084623 },
    { url = "https://files.pythonhosted.org/packages/9b/29/2f57381879747658f0b2cff55eb3964296580a178690e3f672cfee45bc4c/SQLAlchemy-2.0.31-cp311-cp311-macosx_11_0_arm64.whl", hash = "sha256:2e2c38c2a4c5c634fe6c3c58a789712719fa1bf9b9d6ff5ebfce9a9e5b89c1ca", size = 2074995 },
    { url = "https://files.pythonhosted.org/packages/b4/06/9c9b1ddf8ab38d0a550b249e6978582b96181094360e86d2f591b256886b/SQLAlchemy-2.0.31-cp311-cp311-manylinux_2_17_aarch64.manylinux2014_aarch64.whl", hash = "sha256:bd15026f77420eb2b324dcb93551ad9c5f22fab2c150c286ef1dc1160f110203", size = 3193458 },
    { url = "https://files.pythonhosted.org/packages/bb/be/4045425132a2fde3422e0d2062e6a3b874e385b98b0442c1d7d29ac152bc/SQLAlchemy-2.0.31-cp311-cp311-manylinux_2_17_x86_64.manylinux2014_x86_64.whl", hash = "sha256:2196208432deebdfe3b22185d46b08f00ac9d7b01284e168c212919891289396", size = 3193329 },
    { url = "https://files.pythonhosted.org/packages/0b/a9/cce241a226e32ebc71c7b38492e99a8f43b0b8bcacf7c13246cb9e15746a/SQLAlchemy-2.0.31-cp311-cp311-musllinux_1_2_aarch64.whl", hash = "sha256:352b2770097f41bff6029b280c0e03b217c2dcaddc40726f8f53ed58d8a85da4", size = 3132301 },
    { url = "https://files.pythonhosted.org/packages/15/be/6a418dcb048f0896f40b99d371355dcc794dc4cbfc8a262db4305d7cf112/SQLAlchemy-2.0.31-cp311-cp311-musllinux_1_2_x86_64.whl", hash = "sha256:56d51ae825d20d604583f82c9527d285e9e6d14f9a5516463d9705dab20c3740", size = 3151799 },
    { url = "https://files.pythonhosted.org/packages/8a/9d/1e1105848bb75d93749cb6108dc529d242efc84da1e49fa87f17ec296556/SQLAlchemy-2.0.31-cp311-cp311-win32.whl", hash = "sha256:6e2622844551945db81c26a02f27d94145b561f9d4b0c39ce7bfd2fda5776dac", size = 2054824 },
    { url = "https://files.pythonhosted.org/packages/76/c9/6a8f1f512a1663053db9071d4af13a0f4e1d69bedbc4e08f8905415e4eb0/SQLAlchemy-2.0.31-cp311-cp311-win_amd64.whl", hash = "sha256:ccaf1b0c90435b6e430f5dd30a5aede4764942a695552eb3a4ab74ed63c5b8d3", size = 2080602 },
    { url = "https://files.pythonhosted.org/packages/fc/26/1a44b4c9286bccdc2ac344da6487227595512aa60d45c6617b1b6cd859b6/SQLAlchemy-2.0.31-cp312-cp312-macosx_10_9_x86_64.whl", hash = "sha256:3b74570d99126992d4b0f91fb87c586a574a5872651185de8297c6f90055ae42", size = 2083132 },
    { url = "https://files.pythonhosted.org/packages/27/97/21efc51a670e2772156890e241120aac7dae6c92d1bf56e811fb3bec3373/SQLAlchemy-2.0.31-cp312-cp312-macosx_11_0_arm64.whl", hash = "sha256:6f77c4f042ad493cb8595e2f503c7a4fe44cd7bd59c7582fd6d78d7e7b8ec52c", size = 2073619 },
    { url = "https://files.pythonhosted.org/packages/ef/87/778b09abc441b6db4014efb9ef87be0c6959cbd46ee3280faad297f30c12/SQLAlchemy-2.0.31-cp312-cp312-manylinux_2_17_aarch64.manylinux2014_aarch64.whl", hash = "sha256:cd1591329333daf94467e699e11015d9c944f44c94d2091f4ac493ced0119449", size = 3222863 },
    { url = "https://files.pythonhosted.org/packages/60/f8/7794f3055d50e96cee04c19c7faeedacb323c7762f334660ba03bed95485/SQLAlchemy-2.0.31-cp312-cp312-manylinux_2_17_x86_64.manylinux2014_x86_64.whl", hash = "sha256:74afabeeff415e35525bf7a4ecdab015f00e06456166a2eba7590e49f8db940e", size = 3233968 },
    { url = "https://files.pythonhosted.org/packages/e5/ed/aa6ad258a9478874c4b921e8e78d04d237667fc48550a94df79e827ae91f/SQLAlchemy-2.0.31-cp312-cp312-musllinux_1_2_aarch64.whl", hash = "sha256:b9c01990d9015df2c6f818aa8f4297d42ee71c9502026bb074e713d496e26b67", size = 3170897 },
    { url = "https://files.pythonhosted.org/packages/7f/da/d8f8515caf3ef357ad02bb4fd38474fb72c21ee788e2995d9395bcf5185e/SQLAlchemy-2.0.31-cp312-cp312-musllinux_1_2_x86_64.whl", hash = "sha256:66f63278db425838b3c2b1c596654b31939427016ba030e951b292e32b99553e", size = 3197274 },
    { url = "https://files.pythonhosted.org/packages/99/b3/02a2aeb619f5e366b29d9f5eeeaadbe2bd383eab22ce10a00ced870c63f0/SQLAlchemy-2.0.31-cp312-cp312-win32.whl", hash = "sha256:0b0f658414ee4e4b8cbcd4a9bb0fd743c5eeb81fc858ca517217a8013d282c96", size = 2053430 },
    { url = "https://files.pythonhosted.org/packages/d0/ff/5746886a796799d45285573c8a6564b4b84c730142ab974d7a3f7bacee6c/SQLAlchemy-2.0.31-cp312-cp312-win_amd64.whl", hash = "sha256:fa4b1af3e619b5b0b435e333f3967612db06351217c58bfb50cee5f003db2a5a", size = 2079376 },
    { url = "https://files.pythonhosted.org/packages/f3/89/ff21b6c7ccdb254fba5444d15afe193d9a71f4fa054b4823d4384d10718e/SQLAlchemy-2.0.31-py3-none-any.whl", hash = "sha256:69f3e3c08867a8e4856e92d7afb618b95cdee18e0bc1647b77599722c9a28911", size = 1874629 },
]

[[package]]
name = "starlette"
version = "0.37.2"
source = { registry = "https://pypi.org/simple" }
dependencies = [
    { name = "anyio" },
]
sdist = { url = "https://files.pythonhosted.org/packages/61/b5/6bceb93ff20bd7ca36e6f7c540581abb18f53130fabb30ba526e26fd819b/starlette-0.37.2.tar.gz", hash = "sha256:9af890290133b79fc3db55474ade20f6220a364a0402e0b556e7cd5e1e093823", size = 2843736 }
wheels = [
    { url = "https://files.pythonhosted.org/packages/fd/18/31fa32ed6c68ba66220204ef0be798c349d0a20c1901f9d4a794e08c76d8/starlette-0.37.2-py3-none-any.whl", hash = "sha256:6fe59f29268538e5d0d182f2791a479a0c64638e6935d1c6989e63fb2699c6ee", size = 71908 },
]

[[package]]
name = "toolz"
version = "1.0.0"
source = { registry = "https://pypi.org/simple" }
sdist = { url = "https://files.pythonhosted.org/packages/8a/0b/d80dfa675bf592f636d1ea0b835eab4ec8df6e9415d8cfd766df54456123/toolz-1.0.0.tar.gz", hash = "sha256:2c86e3d9a04798ac556793bced838816296a2f085017664e4995cb40a1047a02", size = 66790 }
wheels = [
    { url = "https://files.pythonhosted.org/packages/03/98/eb27cc78ad3af8e302c9d8ff4977f5026676e130d28dd7578132a457170c/toolz-1.0.0-py3-none-any.whl", hash = "sha256:292c8f1c4e7516bf9086f8850935c799a874039c8bcf959d47b600e4c44a6236", size = 56383 },
]

[[package]]
name = "tqdm"
version = "4.66.5"
source = { registry = "https://pypi.org/simple" }
dependencies = [
    { name = "colorama", marker = "platform_system == 'Windows'" },
]
sdist = { url = "https://files.pythonhosted.org/packages/58/83/6ba9844a41128c62e810fddddd72473201f3eacde02046066142a2d96cc5/tqdm-4.66.5.tar.gz", hash = "sha256:e1020aef2e5096702d8a025ac7d16b1577279c9d63f8375b63083e9a5f0fcbad", size = 169504 }
wheels = [
    { url = "https://files.pythonhosted.org/packages/48/5d/acf5905c36149bbaec41ccf7f2b68814647347b72075ac0b1fe3022fdc73/tqdm-4.66.5-py3-none-any.whl", hash = "sha256:90279a3770753eafc9194a0364852159802111925aa30eb3f9d85b0e805ac7cd", size = 78351 },
]

[[package]]
name = "tweepy"
version = "4.14.0"
source = { registry = "https://pypi.org/simple" }
dependencies = [
    { name = "oauthlib" },
    { name = "requests" },
    { name = "requests-oauthlib" },
]
sdist = { url = "https://files.pythonhosted.org/packages/75/1c/0db8c3cf9d31bf63853ff612d201060ae78e6db03468a70e063bef0eda62/tweepy-4.14.0.tar.gz", hash = "sha256:1f9f1707d6972de6cff6c5fd90dfe6a449cd2e0d70bd40043ffab01e07a06c8c", size = 88623 }
wheels = [
    { url = "https://files.pythonhosted.org/packages/4d/78/ba0065d5636bbf4a35b78c4f81b74e7858b609cdf69e629d6da5c91b9d92/tweepy-4.14.0-py3-none-any.whl", hash = "sha256:db6d3844ccc0c6d27f339f12ba8acc89912a961da513c1ae50fa2be502a56afb", size = 98520 },
]

[[package]]
name = "twitter-api-client"
version = "0.10.22"
source = { registry = "https://pypi.org/simple" }
dependencies = [
    { name = "aiofiles" },
    { name = "httpx" },
    { name = "m3u8" },
    { name = "nest-asyncio" },
    { name = "orjson" },
    { name = "tqdm" },
    { name = "uvloop", marker = "platform_system != 'Windows'" },
    { name = "websockets" },
]
sdist = { url = "https://files.pythonhosted.org/packages/4f/3b/234ad943a32dc19c5e516b84dc27921a38a4c2aad5802a2edc63ab4e091f/twitter_api_client-0.10.22.tar.gz", hash = "sha256:4b92b34110c842ba1cd9b26c3cb68a47dc687072a78aa77d674e790ac0b9adb4", size = 44301 }
wheels = [
    { url = "https://files.pythonhosted.org/packages/6c/3b/a15916132cbf17debc4df150e905e8c0082c4a8423e05a49a76801e65110/twitter_api_client-0.10.22-py3-none-any.whl", hash = "sha256:41d672f2f6bc801ff1cff4574428613a7dd4b18fff2ae284d3ad067a33a4ae07", size = 41946 },
]

[[package]]
name = "typer"
version = "0.12.5"
source = { registry = "https://pypi.org/simple" }
dependencies = [
    { name = "click" },
    { name = "rich" },
    { name = "shellingham" },
    { name = "typing-extensions" },
]
sdist = { url = "https://files.pythonhosted.org/packages/c5/58/a79003b91ac2c6890fc5d90145c662fd5771c6f11447f116b63300436bc9/typer-0.12.5.tar.gz", hash = "sha256:f592f089bedcc8ec1b974125d64851029c3b1af145f04aca64d69410f0c9b722", size = 98953 }
wheels = [
    { url = "https://files.pythonhosted.org/packages/a8/2b/886d13e742e514f704c33c4caa7df0f3b89e5a25ef8db02aa9ca3d9535d5/typer-0.12.5-py3-none-any.whl", hash = "sha256:62fe4e471711b147e3365034133904df3e235698399bc4de2b36c8579298d52b", size = 47288 },
]

[[package]]
name = "types-requests"
version = "2.32.0.20241016"
source = { registry = "https://pypi.org/simple" }
dependencies = [
    { name = "urllib3" },
]
sdist = { url = "https://files.pythonhosted.org/packages/fa/3c/4f2a430c01a22abd49a583b6b944173e39e7d01b688190a5618bd59a2e22/types-requests-2.32.0.20241016.tar.gz", hash = "sha256:0d9cad2f27515d0e3e3da7134a1b6f28fb97129d86b867f24d9c726452634d95", size = 18065 }
wheels = [
    { url = "https://files.pythonhosted.org/packages/d7/01/485b3026ff90e5190b5e24f1711522e06c79f4a56c8f4b95848ac072e20f/types_requests-2.32.0.20241016-py3-none-any.whl", hash = "sha256:4195d62d6d3e043a4eaaf08ff8a62184584d2e8684e9d2aa178c7915a7da3747", size = 15836 },
]

[[package]]
name = "typing-extensions"
version = "4.12.2"
source = { registry = "https://pypi.org/simple" }
sdist = { url = "https://files.pythonhosted.org/packages/df/db/f35a00659bc03fec321ba8bce9420de607a1d37f8342eee1863174c69557/typing_extensions-4.12.2.tar.gz", hash = "sha256:1a7ead55c7e559dd4dee8856e3a88b41225abfe1ce8df57b7c13915fe121ffb8", size = 85321 }
wheels = [
    { url = "https://files.pythonhosted.org/packages/26/9f/ad63fc0248c5379346306f8668cda6e2e2e9c95e01216d2b8ffd9ff037d0/typing_extensions-4.12.2-py3-none-any.whl", hash = "sha256:04e5ca0351e0f3f85c6853954072df659d0d13fac324d0072316b67d7794700d", size = 37438 },
]

[[package]]
name = "urllib3"
version = "2.2.3"
source = { registry = "https://pypi.org/simple" }
sdist = { url = "https://files.pythonhosted.org/packages/ed/63/22ba4ebfe7430b76388e7cd448d5478814d3032121827c12a2cc287e2260/urllib3-2.2.3.tar.gz", hash = "sha256:e7d814a81dad81e6caf2ec9fdedb284ecc9c73076b62654547cc64ccdcae26e9", size = 300677 }
wheels = [
    { url = "https://files.pythonhosted.org/packages/ce/d9/5f4c13cecde62396b0d3fe530a50ccea91e7dfc1ccf0e09c228841bb5ba8/urllib3-2.2.3-py3-none-any.whl", hash = "sha256:ca899ca043dcb1bafa3e262d73aa25c465bfb49e0bd9dd5d59f1d0acba2f8fac", size = 126338 },
]

[[package]]
name = "uvicorn"
version = "0.30.3"
source = { registry = "https://pypi.org/simple" }
dependencies = [
    { name = "click" },
    { name = "h11" },
]
sdist = { url = "https://files.pythonhosted.org/packages/77/40/b650be95700dc83d14c5f2b9eac9deb23cbca757a12ee20e473b5ef1ac48/uvicorn-0.30.3.tar.gz", hash = "sha256:0d114d0831ff1adbf231d358cbf42f17333413042552a624ea6a9b4c33dcfd81", size = 42762 }
wheels = [
    { url = "https://files.pythonhosted.org/packages/63/84/2a26b4eac1cf0c6b5b176dd4346cc4912af5e1b0efc150b865e28636ac34/uvicorn-0.30.3-py3-none-any.whl", hash = "sha256:94a3608da0e530cea8f69683aa4126364ac18e3826b6630d1a65f4638aade503", size = 62760 },
]

[package.optional-dependencies]
standard = [
    { name = "colorama", marker = "sys_platform == 'win32'" },
    { name = "httptools" },
    { name = "python-dotenv" },
    { name = "pyyaml" },
    { name = "uvloop", marker = "platform_python_implementation != 'PyPy' and sys_platform != 'cygwin' and sys_platform != 'win32'" },
    { name = "watchfiles" },
    { name = "websockets" },
]

[[package]]
name = "uvloop"
version = "0.21.0"
source = { registry = "https://pypi.org/simple" }
sdist = { url = "https://files.pythonhosted.org/packages/af/c0/854216d09d33c543f12a44b393c402e89a920b1a0a7dc634c42de91b9cf6/uvloop-0.21.0.tar.gz", hash = "sha256:3bf12b0fda68447806a7ad847bfa591613177275d35b6724b1ee573faa3704e3", size = 2492741 }
wheels = [
    { url = "https://files.pythonhosted.org/packages/57/a7/4cf0334105c1160dd6819f3297f8700fda7fc30ab4f61fbf3e725acbc7cc/uvloop-0.21.0-cp311-cp311-macosx_10_9_universal2.whl", hash = "sha256:c0f3fa6200b3108919f8bdabb9a7f87f20e7097ea3c543754cabc7d717d95cf8", size = 1447410 },
    { url = "https://files.pythonhosted.org/packages/8c/7c/1517b0bbc2dbe784b563d6ab54f2ef88c890fdad77232c98ed490aa07132/uvloop-0.21.0-cp311-cp311-macosx_10_9_x86_64.whl", hash = "sha256:0878c2640cf341b269b7e128b1a5fed890adc4455513ca710d77d5e93aa6d6a0", size = 805476 },
    { url = "https://files.pythonhosted.org/packages/ee/ea/0bfae1aceb82a503f358d8d2fa126ca9dbdb2ba9c7866974faec1cb5875c/uvloop-0.21.0-cp311-cp311-manylinux_2_17_aarch64.manylinux2014_aarch64.whl", hash = "sha256:b9fb766bb57b7388745d8bcc53a359b116b8a04c83a2288069809d2b3466c37e", size = 3960855 },
    { url = "https://files.pythonhosted.org/packages/8a/ca/0864176a649838b838f36d44bf31c451597ab363b60dc9e09c9630619d41/uvloop-0.21.0-cp311-cp311-manylinux_2_17_x86_64.manylinux2014_x86_64.whl", hash = "sha256:8a375441696e2eda1c43c44ccb66e04d61ceeffcd76e4929e527b7fa401b90fb", size = 3973185 },
    { url = "https://files.pythonhosted.org/packages/30/bf/08ad29979a936d63787ba47a540de2132169f140d54aa25bc8c3df3e67f4/uvloop-0.21.0-cp311-cp311-musllinux_1_2_aarch64.whl", hash = "sha256:baa0e6291d91649c6ba4ed4b2f982f9fa165b5bbd50a9e203c416a2797bab3c6", size = 3820256 },
    { url = "https://files.pythonhosted.org/packages/da/e2/5cf6ef37e3daf2f06e651aae5ea108ad30df3cb269102678b61ebf1fdf42/uvloop-0.21.0-cp311-cp311-musllinux_1_2_x86_64.whl", hash = "sha256:4509360fcc4c3bd2c70d87573ad472de40c13387f5fda8cb58350a1d7475e58d", size = 3937323 },
    { url = "https://files.pythonhosted.org/packages/8c/4c/03f93178830dc7ce8b4cdee1d36770d2f5ebb6f3d37d354e061eefc73545/uvloop-0.21.0-cp312-cp312-macosx_10_13_universal2.whl", hash = "sha256:359ec2c888397b9e592a889c4d72ba3d6befba8b2bb01743f72fffbde663b59c", size = 1471284 },
    { url = "https://files.pythonhosted.org/packages/43/3e/92c03f4d05e50f09251bd8b2b2b584a2a7f8fe600008bcc4523337abe676/uvloop-0.21.0-cp312-cp312-macosx_10_13_x86_64.whl", hash = "sha256:f7089d2dc73179ce5ac255bdf37c236a9f914b264825fdaacaded6990a7fb4c2", size = 821349 },
    { url = "https://files.pythonhosted.org/packages/a6/ef/a02ec5da49909dbbfb1fd205a9a1ac4e88ea92dcae885e7c961847cd51e2/uvloop-0.21.0-cp312-cp312-manylinux_2_17_aarch64.manylinux2014_aarch64.whl", hash = "sha256:baa4dcdbd9ae0a372f2167a207cd98c9f9a1ea1188a8a526431eef2f8116cc8d", size = 4580089 },
    { url = "https://files.pythonhosted.org/packages/06/a7/b4e6a19925c900be9f98bec0a75e6e8f79bb53bdeb891916609ab3958967/uvloop-0.21.0-cp312-cp312-manylinux_2_17_x86_64.manylinux2014_x86_64.whl", hash = "sha256:86975dca1c773a2c9864f4c52c5a55631038e387b47eaf56210f873887b6c8dc", size = 4693770 },
    { url = "https://files.pythonhosted.org/packages/ce/0c/f07435a18a4b94ce6bd0677d8319cd3de61f3a9eeb1e5f8ab4e8b5edfcb3/uvloop-0.21.0-cp312-cp312-musllinux_1_2_aarch64.whl", hash = "sha256:461d9ae6660fbbafedd07559c6a2e57cd553b34b0065b6550685f6653a98c1cb", size = 4451321 },
    { url = "https://files.pythonhosted.org/packages/8f/eb/f7032be105877bcf924709c97b1bf3b90255b4ec251f9340cef912559f28/uvloop-0.21.0-cp312-cp312-musllinux_1_2_x86_64.whl", hash = "sha256:183aef7c8730e54c9a3ee3227464daed66e37ba13040bb3f350bc2ddc040f22f", size = 4659022 },
    { url = "https://files.pythonhosted.org/packages/3f/8d/2cbef610ca21539f0f36e2b34da49302029e7c9f09acef0b1c3b5839412b/uvloop-0.21.0-cp313-cp313-macosx_10_13_universal2.whl", hash = "sha256:bfd55dfcc2a512316e65f16e503e9e450cab148ef11df4e4e679b5e8253a5281", size = 1468123 },
    { url = "https://files.pythonhosted.org/packages/93/0d/b0038d5a469f94ed8f2b2fce2434a18396d8fbfb5da85a0a9781ebbdec14/uvloop-0.21.0-cp313-cp313-macosx_10_13_x86_64.whl", hash = "sha256:787ae31ad8a2856fc4e7c095341cccc7209bd657d0e71ad0dc2ea83c4a6fa8af", size = 819325 },
    { url = "https://files.pythonhosted.org/packages/50/94/0a687f39e78c4c1e02e3272c6b2ccdb4e0085fda3b8352fecd0410ccf915/uvloop-0.21.0-cp313-cp313-manylinux_2_17_aarch64.manylinux2014_aarch64.whl", hash = "sha256:5ee4d4ef48036ff6e5cfffb09dd192c7a5027153948d85b8da7ff705065bacc6", size = 4582806 },
    { url = "https://files.pythonhosted.org/packages/d2/19/f5b78616566ea68edd42aacaf645adbf71fbd83fc52281fba555dc27e3f1/uvloop-0.21.0-cp313-cp313-manylinux_2_17_x86_64.manylinux2014_x86_64.whl", hash = "sha256:f3df876acd7ec037a3d005b3ab85a7e4110422e4d9c1571d4fc89b0fc41b6816", size = 4701068 },
    { url = "https://files.pythonhosted.org/packages/47/57/66f061ee118f413cd22a656de622925097170b9380b30091b78ea0c6ea75/uvloop-0.21.0-cp313-cp313-musllinux_1_2_aarch64.whl", hash = "sha256:bd53ecc9a0f3d87ab847503c2e1552b690362e005ab54e8a48ba97da3924c0dc", size = 4454428 },
    { url = "https://files.pythonhosted.org/packages/63/9a/0962b05b308494e3202d3f794a6e85abe471fe3cafdbcf95c2e8c713aabd/uvloop-0.21.0-cp313-cp313-musllinux_1_2_x86_64.whl", hash = "sha256:a5c39f217ab3c663dc699c04cbd50c13813e31d917642d459fdcec07555cc553", size = 4660018 },
]

[[package]]
name = "watchfiles"
version = "0.24.0"
source = { registry = "https://pypi.org/simple" }
dependencies = [
    { name = "anyio" },
]
sdist = { url = "https://files.pythonhosted.org/packages/c8/27/2ba23c8cc85796e2d41976439b08d52f691655fdb9401362099502d1f0cf/watchfiles-0.24.0.tar.gz", hash = "sha256:afb72325b74fa7a428c009c1b8be4b4d7c2afedafb2982827ef2156646df2fe1", size = 37870 }
wheels = [
    { url = "https://files.pythonhosted.org/packages/85/02/366ae902cd81ca5befcd1854b5c7477b378f68861597cef854bd6dc69fbe/watchfiles-0.24.0-cp311-cp311-macosx_10_12_x86_64.whl", hash = "sha256:bdcd5538e27f188dd3c804b4a8d5f52a7fc7f87e7fd6b374b8e36a4ca03db428", size = 375579 },
    { url = "https://files.pythonhosted.org/packages/bc/67/d8c9d256791fe312fea118a8a051411337c948101a24586e2df237507976/watchfiles-0.24.0-cp311-cp311-macosx_11_0_arm64.whl", hash = "sha256:2dadf8a8014fde6addfd3c379e6ed1a981c8f0a48292d662e27cabfe4239c83c", size = 367726 },
    { url = "https://files.pythonhosted.org/packages/b1/dc/a8427b21ef46386adf824a9fec4be9d16a475b850616cfd98cf09a97a2ef/watchfiles-0.24.0-cp311-cp311-manylinux_2_17_aarch64.manylinux2014_aarch64.whl", hash = "sha256:6509ed3f467b79d95fc62a98229f79b1a60d1b93f101e1c61d10c95a46a84f43", size = 437735 },
    { url = "https://files.pythonhosted.org/packages/3a/21/0b20bef581a9fbfef290a822c8be645432ceb05fb0741bf3c032e0d90d9a/watchfiles-0.24.0-cp311-cp311-manylinux_2_17_armv7l.manylinux2014_armv7l.whl", hash = "sha256:8360f7314a070c30e4c976b183d1d8d1585a4a50c5cb603f431cebcbb4f66327", size = 433644 },
    { url = "https://files.pythonhosted.org/packages/1c/e8/d5e5f71cc443c85a72e70b24269a30e529227986096abe091040d6358ea9/watchfiles-0.24.0-cp311-cp311-manylinux_2_17_i686.manylinux2014_i686.whl", hash = "sha256:316449aefacf40147a9efaf3bd7c9bdd35aaba9ac5d708bd1eb5763c9a02bef5", size = 450928 },
    { url = "https://files.pythonhosted.org/packages/61/ee/bf17f5a370c2fcff49e1fec987a6a43fd798d8427ea754ce45b38f9e117a/watchfiles-0.24.0-cp311-cp311-manylinux_2_17_ppc64le.manylinux2014_ppc64le.whl", hash = "sha256:73bde715f940bea845a95247ea3e5eb17769ba1010efdc938ffcb967c634fa61", size = 469072 },
    { url = "https://files.pythonhosted.org/packages/a3/34/03b66d425986de3fc6077e74a74c78da298f8cb598887f664a4485e55543/watchfiles-0.24.0-cp311-cp311-manylinux_2_17_s390x.manylinux2014_s390x.whl", hash = "sha256:3770e260b18e7f4e576edca4c0a639f704088602e0bc921c5c2e721e3acb8d15", size = 475517 },
    { url = "https://files.pythonhosted.org/packages/70/eb/82f089c4f44b3171ad87a1b433abb4696f18eb67292909630d886e073abe/watchfiles-0.24.0-cp311-cp311-manylinux_2_17_x86_64.manylinux2014_x86_64.whl", hash = "sha256:aa0fd7248cf533c259e59dc593a60973a73e881162b1a2f73360547132742823", size = 425480 },
    { url = "https://files.pythonhosted.org/packages/53/20/20509c8f5291e14e8a13104b1808cd7cf5c44acd5feaecb427a49d387774/watchfiles-0.24.0-cp311-cp311-musllinux_1_1_aarch64.whl", hash = "sha256:d7a2e3b7f5703ffbd500dabdefcbc9eafeff4b9444bbdd5d83d79eedf8428fab", size = 612322 },
    { url = "https://files.pythonhosted.org/packages/df/2b/5f65014a8cecc0a120f5587722068a975a692cadbe9fe4ea56b3d8e43f14/watchfiles-0.24.0-cp311-cp311-musllinux_1_1_x86_64.whl", hash = "sha256:d831ee0a50946d24a53821819b2327d5751b0c938b12c0653ea5be7dea9c82ec", size = 595094 },
    { url = "https://files.pythonhosted.org/packages/18/98/006d8043a82c0a09d282d669c88e587b3a05cabdd7f4900e402250a249ac/watchfiles-0.24.0-cp311-none-win32.whl", hash = "sha256:49d617df841a63b4445790a254013aea2120357ccacbed00253f9c2b5dc24e2d", size = 264191 },
    { url = "https://files.pythonhosted.org/packages/8a/8b/badd9247d6ec25f5f634a9b3d0d92e39c045824ec7e8afcedca8ee52c1e2/watchfiles-0.24.0-cp311-none-win_amd64.whl", hash = "sha256:d3dcb774e3568477275cc76554b5a565024b8ba3a0322f77c246bc7111c5bb9c", size = 277527 },
    { url = "https://files.pythonhosted.org/packages/af/19/35c957c84ee69d904299a38bae3614f7cede45f07f174f6d5a2f4dbd6033/watchfiles-0.24.0-cp311-none-win_arm64.whl", hash = "sha256:9301c689051a4857d5b10777da23fafb8e8e921bcf3abe6448a058d27fb67633", size = 266253 },
    { url = "https://files.pythonhosted.org/packages/35/82/92a7bb6dc82d183e304a5f84ae5437b59ee72d48cee805a9adda2488b237/watchfiles-0.24.0-cp312-cp312-macosx_10_12_x86_64.whl", hash = "sha256:7211b463695d1e995ca3feb38b69227e46dbd03947172585ecb0588f19b0d87a", size = 374137 },
    { url = "https://files.pythonhosted.org/packages/87/91/49e9a497ddaf4da5e3802d51ed67ff33024597c28f652b8ab1e7c0f5718b/watchfiles-0.24.0-cp312-cp312-macosx_11_0_arm64.whl", hash = "sha256:4b8693502d1967b00f2fb82fc1e744df128ba22f530e15b763c8d82baee15370", size = 367733 },
    { url = "https://files.pythonhosted.org/packages/0d/d8/90eb950ab4998effea2df4cf3a705dc594f6bc501c5a353073aa990be965/watchfiles-0.24.0-cp312-cp312-manylinux_2_17_aarch64.manylinux2014_aarch64.whl", hash = "sha256:cdab9555053399318b953a1fe1f586e945bc8d635ce9d05e617fd9fe3a4687d6", size = 437322 },
    { url = "https://files.pythonhosted.org/packages/6c/a2/300b22e7bc2a222dd91fce121cefa7b49aa0d26a627b2777e7bdfcf1110b/watchfiles-0.24.0-cp312-cp312-manylinux_2_17_armv7l.manylinux2014_armv7l.whl", hash = "sha256:34e19e56d68b0dad5cff62273107cf5d9fbaf9d75c46277aa5d803b3ef8a9e9b", size = 433409 },
    { url = "https://files.pythonhosted.org/packages/99/44/27d7708a43538ed6c26708bcccdde757da8b7efb93f4871d4cc39cffa1cc/watchfiles-0.24.0-cp312-cp312-manylinux_2_17_i686.manylinux2014_i686.whl", hash = "sha256:41face41f036fee09eba33a5b53a73e9a43d5cb2c53dad8e61fa6c9f91b5a51e", size = 452142 },
    { url = "https://files.pythonhosted.org/packages/b0/ec/c4e04f755be003129a2c5f3520d2c47026f00da5ecb9ef1e4f9449637571/watchfiles-0.24.0-cp312-cp312-manylinux_2_17_ppc64le.manylinux2014_ppc64le.whl", hash = "sha256:5148c2f1ea043db13ce9b0c28456e18ecc8f14f41325aa624314095b6aa2e9ea", size = 469414 },
    { url = "https://files.pythonhosted.org/packages/c5/4e/cdd7de3e7ac6432b0abf282ec4c1a1a2ec62dfe423cf269b86861667752d/watchfiles-0.24.0-cp312-cp312-manylinux_2_17_s390x.manylinux2014_s390x.whl", hash = "sha256:7e4bd963a935aaf40b625c2499f3f4f6bbd0c3776f6d3bc7c853d04824ff1c9f", size = 472962 },
    { url = "https://files.pythonhosted.org/packages/27/69/e1da9d34da7fc59db358424f5d89a56aaafe09f6961b64e36457a80a7194/watchfiles-0.24.0-cp312-cp312-manylinux_2_17_x86_64.manylinux2014_x86_64.whl", hash = "sha256:c79d7719d027b7a42817c5d96461a99b6a49979c143839fc37aa5748c322f234", size = 425705 },
    { url = "https://files.pythonhosted.org/packages/e8/c1/24d0f7357be89be4a43e0a656259676ea3d7a074901f47022f32e2957798/watchfiles-0.24.0-cp312-cp312-musllinux_1_1_aarch64.whl", hash = "sha256:32aa53a9a63b7f01ed32e316e354e81e9da0e6267435c7243bf8ae0f10b428ef", size = 612851 },
    { url = "https://files.pythonhosted.org/packages/c7/af/175ba9b268dec56f821639c9893b506c69fd999fe6a2e2c51de420eb2f01/watchfiles-0.24.0-cp312-cp312-musllinux_1_1_x86_64.whl", hash = "sha256:ce72dba6a20e39a0c628258b5c308779b8697f7676c254a845715e2a1039b968", size = 594868 },
    { url = "https://files.pythonhosted.org/packages/44/81/1f701323a9f70805bc81c74c990137123344a80ea23ab9504a99492907f8/watchfiles-0.24.0-cp312-none-win32.whl", hash = "sha256:d9018153cf57fc302a2a34cb7564870b859ed9a732d16b41a9b5cb2ebed2d444", size = 264109 },
    { url = "https://files.pythonhosted.org/packages/b4/0b/32cde5bc2ebd9f351be326837c61bdeb05ad652b793f25c91cac0b48a60b/watchfiles-0.24.0-cp312-none-win_amd64.whl", hash = "sha256:551ec3ee2a3ac9cbcf48a4ec76e42c2ef938a7e905a35b42a1267fa4b1645896", size = 277055 },
    { url = "https://files.pythonhosted.org/packages/4b/81/daade76ce33d21dbec7a15afd7479de8db786e5f7b7d249263b4ea174e08/watchfiles-0.24.0-cp312-none-win_arm64.whl", hash = "sha256:b52a65e4ea43c6d149c5f8ddb0bef8d4a1e779b77591a458a893eb416624a418", size = 266169 },
    { url = "https://files.pythonhosted.org/packages/30/dc/6e9f5447ae14f645532468a84323a942996d74d5e817837a5c8ce9d16c69/watchfiles-0.24.0-cp313-cp313-macosx_10_12_x86_64.whl", hash = "sha256:3d2e3ab79a1771c530233cadfd277fcc762656d50836c77abb2e5e72b88e3a48", size = 373764 },
    { url = "https://files.pythonhosted.org/packages/79/c0/c3a9929c372816c7fc87d8149bd722608ea58dc0986d3ef7564c79ad7112/watchfiles-0.24.0-cp313-cp313-macosx_11_0_arm64.whl", hash = "sha256:327763da824817b38ad125dcd97595f942d720d32d879f6c4ddf843e3da3fe90", size = 367873 },
    { url = "https://files.pythonhosted.org/packages/2e/11/ff9a4445a7cfc1c98caf99042df38964af12eed47d496dd5d0d90417349f/watchfiles-0.24.0-cp313-cp313-manylinux_2_17_aarch64.manylinux2014_aarch64.whl", hash = "sha256:bd82010f8ab451dabe36054a1622870166a67cf3fce894f68895db6f74bbdc94", size = 438381 },
    { url = "https://files.pythonhosted.org/packages/48/a3/763ba18c98211d7bb6c0f417b2d7946d346cdc359d585cc28a17b48e964b/watchfiles-0.24.0-cp313-cp313-manylinux_2_17_armv7l.manylinux2014_armv7l.whl", hash = "sha256:d64ba08db72e5dfd5c33be1e1e687d5e4fcce09219e8aee893a4862034081d4e", size = 432809 },
    { url = "https://files.pythonhosted.org/packages/30/4c/616c111b9d40eea2547489abaf4ffc84511e86888a166d3a4522c2ba44b5/watchfiles-0.24.0-cp313-cp313-manylinux_2_17_i686.manylinux2014_i686.whl", hash = "sha256:1cf1f6dd7825053f3d98f6d33f6464ebdd9ee95acd74ba2c34e183086900a827", size = 451801 },
    { url = "https://files.pythonhosted.org/packages/b6/be/d7da83307863a422abbfeb12903a76e43200c90ebe5d6afd6a59d158edea/watchfiles-0.24.0-cp313-cp313-manylinux_2_17_ppc64le.manylinux2014_ppc64le.whl", hash = "sha256:43e3e37c15a8b6fe00c1bce2473cfa8eb3484bbeecf3aefbf259227e487a03df", size = 468886 },
    { url = "https://files.pythonhosted.org/packages/1d/d3/3dfe131ee59d5e90b932cf56aba5c996309d94dafe3d02d204364c23461c/watchfiles-0.24.0-cp313-cp313-manylinux_2_17_s390x.manylinux2014_s390x.whl", hash = "sha256:88bcd4d0fe1d8ff43675360a72def210ebad3f3f72cabfeac08d825d2639b4ab", size = 472973 },
    { url = "https://files.pythonhosted.org/packages/42/6c/279288cc5653a289290d183b60a6d80e05f439d5bfdfaf2d113738d0f932/watchfiles-0.24.0-cp313-cp313-manylinux_2_17_x86_64.manylinux2014_x86_64.whl", hash = "sha256:999928c6434372fde16c8f27143d3e97201160b48a614071261701615a2a156f", size = 425282 },
    { url = "https://files.pythonhosted.org/packages/d6/d7/58afe5e85217e845edf26d8780c2d2d2ae77675eeb8d1b8b8121d799ce52/watchfiles-0.24.0-cp313-cp313-musllinux_1_1_aarch64.whl", hash = "sha256:30bbd525c3262fd9f4b1865cb8d88e21161366561cd7c9e1194819e0a33ea86b", size = 612540 },
    { url = "https://files.pythonhosted.org/packages/6d/d5/b96eeb9fe3fda137200dd2f31553670cbc731b1e13164fd69b49870b76ec/watchfiles-0.24.0-cp313-cp313-musllinux_1_1_x86_64.whl", hash = "sha256:edf71b01dec9f766fb285b73930f95f730bb0943500ba0566ae234b5c1618c18", size = 593625 },
    { url = "https://files.pythonhosted.org/packages/c1/e5/c326fe52ee0054107267608d8cea275e80be4455b6079491dfd9da29f46f/watchfiles-0.24.0-cp313-none-win32.whl", hash = "sha256:f4c96283fca3ee09fb044f02156d9570d156698bc3734252175a38f0e8975f07", size = 263899 },
    { url = "https://files.pythonhosted.org/packages/a6/8b/8a7755c5e7221bb35fe4af2dc44db9174f90ebf0344fd5e9b1e8b42d381e/watchfiles-0.24.0-cp313-none-win_amd64.whl", hash = "sha256:a974231b4fdd1bb7f62064a0565a6b107d27d21d9acb50c484d2cdba515b9366", size = 276622 },
]

[[package]]
name = "web3"
version = "7.4.0"
source = { registry = "https://pypi.org/simple" }
dependencies = [
    { name = "aiohttp" },
    { name = "eth-abi" },
    { name = "eth-account" },
    { name = "eth-hash", extra = ["pycryptodome"] },
    { name = "eth-typing" },
    { name = "eth-utils" },
    { name = "hexbytes" },
    { name = "pydantic" },
    { name = "pyunormalize" },
    { name = "pywin32", marker = "platform_system == 'Windows'" },
    { name = "requests" },
    { name = "types-requests" },
    { name = "typing-extensions" },
    { name = "websockets" },
]
sdist = { url = "https://files.pythonhosted.org/packages/a9/0d/95473eb6a18945dee372377fb0c822c1b8e10f1cb946d5e61ac0c201f182/web3-7.4.0.tar.gz", hash = "sha256:473e85ee9ff61f20acb114756bbf222f2f75b42daedc1c3e9183ce6431c6039e", size = 2160903 }
wheels = [
    { url = "https://files.pythonhosted.org/packages/54/b6/8d7da7e76a5ff3e2d6f0699a430dcc753c634bb8de73c558bf4c2a0186c9/web3-7.4.0-py3-none-any.whl", hash = "sha256:09864c549e0ab97a9915f0c02a98f9e87bc1bfb23fedc3e4e6a95e5ab11787b6", size = 1344133 },
]

[[package]]
name = "websockets"
version = "13.1"
source = { registry = "https://pypi.org/simple" }
sdist = { url = "https://files.pythonhosted.org/packages/e2/73/9223dbc7be3dcaf2a7bbf756c351ec8da04b1fa573edaf545b95f6b0c7fd/websockets-13.1.tar.gz", hash = "sha256:a3b3366087c1bc0a2795111edcadddb8b3b59509d5db5d7ea3fdd69f954a8878", size = 158549 }
wheels = [
    { url = "https://files.pythonhosted.org/packages/b2/f0/cf0b8a30d86b49e267ac84addbebbc7a48a6e7bb7c19db80f62411452311/websockets-13.1-cp311-cp311-macosx_10_9_universal2.whl", hash = "sha256:61fc0dfcda609cda0fc9fe7977694c0c59cf9d749fbb17f4e9483929e3c48a19", size = 157813 },
    { url = "https://files.pythonhosted.org/packages/bf/e7/22285852502e33071a8cf0ac814f8988480ec6db4754e067b8b9d0e92498/websockets-13.1-cp311-cp311-macosx_10_9_x86_64.whl", hash = "sha256:ceec59f59d092c5007e815def4ebb80c2de330e9588e101cf8bd94c143ec78a5", size = 155469 },
    { url = "https://files.pythonhosted.org/packages/68/d4/c8c7c1e5b40ee03c5cc235955b0fb1ec90e7e37685a5f69229ad4708dcde/websockets-13.1-cp311-cp311-macosx_11_0_arm64.whl", hash = "sha256:c1dca61c6db1166c48b95198c0b7d9c990b30c756fc2923cc66f68d17dc558fd", size = 155717 },
    { url = "https://files.pythonhosted.org/packages/c9/e4/c50999b9b848b1332b07c7fd8886179ac395cb766fda62725d1539e7bc6c/websockets-13.1-cp311-cp311-manylinux_2_17_aarch64.manylinux2014_aarch64.whl", hash = "sha256:308e20f22c2c77f3f39caca508e765f8725020b84aa963474e18c59accbf4c02", size = 165379 },
    { url = "https://files.pythonhosted.org/packages/bc/49/4a4ad8c072f18fd79ab127650e47b160571aacfc30b110ee305ba25fffc9/websockets-13.1-cp311-cp311-manylinux_2_5_i686.manylinux1_i686.manylinux_2_17_i686.manylinux2014_i686.whl", hash = "sha256:62d516c325e6540e8a57b94abefc3459d7dab8ce52ac75c96cad5549e187e3a7", size = 164376 },
    { url = "https://files.pythonhosted.org/packages/af/9b/8c06d425a1d5a74fd764dd793edd02be18cf6fc3b1ccd1f29244ba132dc0/websockets-13.1-cp311-cp311-manylinux_2_5_x86_64.manylinux1_x86_64.manylinux_2_17_x86_64.manylinux2014_x86_64.whl", hash = "sha256:87c6e35319b46b99e168eb98472d6c7d8634ee37750d7693656dc766395df096", size = 164753 },
    { url = "https://files.pythonhosted.org/packages/d5/5b/0acb5815095ff800b579ffc38b13ab1b915b317915023748812d24e0c1ac/websockets-13.1-cp311-cp311-musllinux_1_2_aarch64.whl", hash = "sha256:5f9fee94ebafbc3117c30be1844ed01a3b177bb6e39088bc6b2fa1dc15572084", size = 165051 },
    { url = "https://files.pythonhosted.org/packages/30/93/c3891c20114eacb1af09dedfcc620c65c397f4fd80a7009cd12d9457f7f5/websockets-13.1-cp311-cp311-musllinux_1_2_i686.whl", hash = "sha256:7c1e90228c2f5cdde263253fa5db63e6653f1c00e7ec64108065a0b9713fa1b3", size = 164489 },
    { url = "https://files.pythonhosted.org/packages/28/09/af9e19885539759efa2e2cd29b8b3f9eecef7ecefea40d46612f12138b36/websockets-13.1-cp311-cp311-musllinux_1_2_x86_64.whl", hash = "sha256:6548f29b0e401eea2b967b2fdc1c7c7b5ebb3eeb470ed23a54cd45ef078a0db9", size = 164438 },
    { url = "https://files.pythonhosted.org/packages/b6/08/6f38b8e625b3d93de731f1d248cc1493327f16cb45b9645b3e791782cff0/websockets-13.1-cp311-cp311-win32.whl", hash = "sha256:c11d4d16e133f6df8916cc5b7e3e96ee4c44c936717d684a94f48f82edb7c92f", size = 158710 },
    { url = "https://files.pythonhosted.org/packages/fb/39/ec8832ecb9bb04a8d318149005ed8cee0ba4e0205835da99e0aa497a091f/websockets-13.1-cp311-cp311-win_amd64.whl", hash = "sha256:d04f13a1d75cb2b8382bdc16ae6fa58c97337253826dfe136195b7f89f661557", size = 159137 },
    { url = "https://files.pythonhosted.org/packages/df/46/c426282f543b3c0296cf964aa5a7bb17e984f58dde23460c3d39b3148fcf/websockets-13.1-cp312-cp312-macosx_10_9_universal2.whl", hash = "sha256:9d75baf00138f80b48f1eac72ad1535aac0b6461265a0bcad391fc5aba875cfc", size = 157821 },
    { url = "https://files.pythonhosted.org/packages/aa/85/22529867010baac258da7c45848f9415e6cf37fef00a43856627806ffd04/websockets-13.1-cp312-cp312-macosx_10_9_x86_64.whl", hash = "sha256:9b6f347deb3dcfbfde1c20baa21c2ac0751afaa73e64e5b693bb2b848efeaa49", size = 155480 },
    { url = "https://files.pythonhosted.org/packages/29/2c/bdb339bfbde0119a6e84af43ebf6275278698a2241c2719afc0d8b0bdbf2/websockets-13.1-cp312-cp312-macosx_11_0_arm64.whl", hash = "sha256:de58647e3f9c42f13f90ac7e5f58900c80a39019848c5547bc691693098ae1bd", size = 155715 },
    { url = "https://files.pythonhosted.org/packages/9f/d0/8612029ea04c5c22bf7af2fd3d63876c4eaeef9b97e86c11972a43aa0e6c/websockets-13.1-cp312-cp312-manylinux_2_17_aarch64.manylinux2014_aarch64.whl", hash = "sha256:a1b54689e38d1279a51d11e3467dd2f3a50f5f2e879012ce8f2d6943f00e83f0", size = 165647 },
    { url = "https://files.pythonhosted.org/packages/56/04/1681ed516fa19ca9083f26d3f3a302257e0911ba75009533ed60fbb7b8d1/websockets-13.1-cp312-cp312-manylinux_2_5_i686.manylinux1_i686.manylinux_2_17_i686.manylinux2014_i686.whl", hash = "sha256:cf1781ef73c073e6b0f90af841aaf98501f975d306bbf6221683dd594ccc52b6", size = 164592 },
    { url = "https://files.pythonhosted.org/packages/38/6f/a96417a49c0ed132bb6087e8e39a37db851c70974f5c724a4b2a70066996/websockets-13.1-cp312-cp312-manylinux_2_5_x86_64.manylinux1_x86_64.manylinux_2_17_x86_64.manylinux2014_x86_64.whl", hash = "sha256:8d23b88b9388ed85c6faf0e74d8dec4f4d3baf3ecf20a65a47b836d56260d4b9", size = 165012 },
    { url = "https://files.pythonhosted.org/packages/40/8b/fccf294919a1b37d190e86042e1a907b8f66cff2b61e9befdbce03783e25/websockets-13.1-cp312-cp312-musllinux_1_2_aarch64.whl", hash = "sha256:3c78383585f47ccb0fcf186dcb8a43f5438bd7d8f47d69e0b56f71bf431a0a68", size = 165311 },
    { url = "https://files.pythonhosted.org/packages/c1/61/f8615cf7ce5fe538476ab6b4defff52beb7262ff8a73d5ef386322d9761d/websockets-13.1-cp312-cp312-musllinux_1_2_i686.whl", hash = "sha256:d6d300f8ec35c24025ceb9b9019ae9040c1ab2f01cddc2bcc0b518af31c75c14", size = 164692 },
    { url = "https://files.pythonhosted.org/packages/5c/f1/a29dd6046d3a722d26f182b783a7997d25298873a14028c4760347974ea3/websockets-13.1-cp312-cp312-musllinux_1_2_x86_64.whl", hash = "sha256:a9dcaf8b0cc72a392760bb8755922c03e17a5a54e08cca58e8b74f6902b433cf", size = 164686 },
    { url = "https://files.pythonhosted.org/packages/0f/99/ab1cdb282f7e595391226f03f9b498f52109d25a2ba03832e21614967dfa/websockets-13.1-cp312-cp312-win32.whl", hash = "sha256:2f85cf4f2a1ba8f602298a853cec8526c2ca42a9a4b947ec236eaedb8f2dc80c", size = 158712 },
    { url = "https://files.pythonhosted.org/packages/46/93/e19160db48b5581feac8468330aa11b7292880a94a37d7030478596cc14e/websockets-13.1-cp312-cp312-win_amd64.whl", hash = "sha256:38377f8b0cdeee97c552d20cf1865695fcd56aba155ad1b4ca8779a5b6ef4ac3", size = 159145 },
    { url = "https://files.pythonhosted.org/packages/51/20/2b99ca918e1cbd33c53db2cace5f0c0cd8296fc77558e1908799c712e1cd/websockets-13.1-cp313-cp313-macosx_10_13_universal2.whl", hash = "sha256:a9ab1e71d3d2e54a0aa646ab6d4eebfaa5f416fe78dfe4da2839525dc5d765c6", size = 157828 },
    { url = "https://files.pythonhosted.org/packages/b8/47/0932a71d3d9c0e9483174f60713c84cee58d62839a143f21a2bcdbd2d205/websockets-13.1-cp313-cp313-macosx_10_13_x86_64.whl", hash = "sha256:b9d7439d7fab4dce00570bb906875734df13d9faa4b48e261c440a5fec6d9708", size = 155487 },
    { url = "https://files.pythonhosted.org/packages/a9/60/f1711eb59ac7a6c5e98e5637fef5302f45b6f76a2c9d64fd83bbb341377a/websockets-13.1-cp313-cp313-macosx_11_0_arm64.whl", hash = "sha256:327b74e915cf13c5931334c61e1a41040e365d380f812513a255aa804b183418", size = 155721 },
    { url = "https://files.pythonhosted.org/packages/6a/e6/ba9a8db7f9d9b0e5f829cf626ff32677f39824968317223605a6b419d445/websockets-13.1-cp313-cp313-manylinux_2_17_aarch64.manylinux2014_aarch64.whl", hash = "sha256:325b1ccdbf5e5725fdcb1b0e9ad4d2545056479d0eee392c291c1bf76206435a", size = 165609 },
    { url = "https://files.pythonhosted.org/packages/c1/22/4ec80f1b9c27a0aebd84ccd857252eda8418ab9681eb571b37ca4c5e1305/websockets-13.1-cp313-cp313-manylinux_2_5_i686.manylinux1_i686.manylinux_2_17_i686.manylinux2014_i686.whl", hash = "sha256:346bee67a65f189e0e33f520f253d5147ab76ae42493804319b5716e46dddf0f", size = 164556 },
    { url = "https://files.pythonhosted.org/packages/27/ac/35f423cb6bb15600438db80755609d27eda36d4c0b3c9d745ea12766c45e/websockets-13.1-cp313-cp313-manylinux_2_5_x86_64.manylinux1_x86_64.manylinux_2_17_x86_64.manylinux2014_x86_64.whl", hash = "sha256:91a0fa841646320ec0d3accdff5b757b06e2e5c86ba32af2e0815c96c7a603c5", size = 164993 },
    { url = "https://files.pythonhosted.org/packages/31/4e/98db4fd267f8be9e52e86b6ee4e9aa7c42b83452ea0ea0672f176224b977/websockets-13.1-cp313-cp313-musllinux_1_2_aarch64.whl", hash = "sha256:18503d2c5f3943e93819238bf20df71982d193f73dcecd26c94514f417f6b135", size = 165360 },
    { url = "https://files.pythonhosted.org/packages/3f/15/3f0de7cda70ffc94b7e7024544072bc5b26e2c1eb36545291abb755d8cdb/websockets-13.1-cp313-cp313-musllinux_1_2_i686.whl", hash = "sha256:a9cd1af7e18e5221d2878378fbc287a14cd527fdd5939ed56a18df8a31136bb2", size = 164745 },
    { url = "https://files.pythonhosted.org/packages/a1/6e/66b6b756aebbd680b934c8bdbb6dcb9ce45aad72cde5f8a7208dbb00dd36/websockets-13.1-cp313-cp313-musllinux_1_2_x86_64.whl", hash = "sha256:70c5be9f416aa72aab7a2a76c90ae0a4fe2755c1816c153c1a2bcc3333ce4ce6", size = 164732 },
    { url = "https://files.pythonhosted.org/packages/35/c6/12e3aab52c11aeb289e3dbbc05929e7a9d90d7a9173958477d3ef4f8ce2d/websockets-13.1-cp313-cp313-win32.whl", hash = "sha256:624459daabeb310d3815b276c1adef475b3e6804abaf2d9d2c061c319f7f187d", size = 158709 },
    { url = "https://files.pythonhosted.org/packages/41/d8/63d6194aae711d7263df4498200c690a9c39fb437ede10f3e157a6343e0d/websockets-13.1-cp313-cp313-win_amd64.whl", hash = "sha256:c518e84bb59c2baae725accd355c8dc517b4a3ed8db88b4bc93c78dae2974bf2", size = 159144 },
    { url = "https://files.pythonhosted.org/packages/56/27/96a5cd2626d11c8280656c6c71d8ab50fe006490ef9971ccd154e0c42cd2/websockets-13.1-py3-none-any.whl", hash = "sha256:a9a396a6ad26130cdae92ae10c36af09d9bfe6cafe69670fd3b6da9b07b4044f", size = 152134 },
]

[[package]]
name = "yarl"
version = "1.16.0"
source = { registry = "https://pypi.org/simple" }
dependencies = [
    { name = "idna" },
    { name = "multidict" },
    { name = "propcache" },
]
sdist = { url = "https://files.pythonhosted.org/packages/23/52/e9766cc6c2eab7dd1e9749c52c9879317500b46fb97d4105223f86679f93/yarl-1.16.0.tar.gz", hash = "sha256:b6f687ced5510a9a2474bbae96a4352e5ace5fa34dc44a217b0537fec1db00b4", size = 176548 }
wheels = [
    { url = "https://files.pythonhosted.org/packages/0a/00/b29affe83de95e403f8a2a669b5a33f1e7dfe686264008100052eb0b05fd/yarl-1.16.0-cp311-cp311-macosx_10_9_universal2.whl", hash = "sha256:d8643975a0080f361639787415a038bfc32d29208a4bf6b783ab3075a20b1ef3", size = 140120 },
    { url = "https://files.pythonhosted.org/packages/3f/22/bcc9799950281a5d4f646536854839ccdbb965e900827ef0750680f81faf/yarl-1.16.0-cp311-cp311-macosx_10_9_x86_64.whl", hash = "sha256:676d96bafc8c2d0039cea0cd3fd44cee7aa88b8185551a2bb93354668e8315c2", size = 92956 },
    { url = "https://files.pythonhosted.org/packages/33/0f/1b76d853d9d921d68bd9991648be17d34e7ac51e2e20e7658f8ee7e2e2ad/yarl-1.16.0-cp311-cp311-macosx_11_0_arm64.whl", hash = "sha256:d9525f03269e64310416dbe6c68d3b23e5d34aaa8f47193a1c45ac568cecbc49", size = 90891 },
    { url = "https://files.pythonhosted.org/packages/61/19/3666d990c24aae98c748e2c262adc9b3a71e38834df007ac5317f4bbd789/yarl-1.16.0-cp311-cp311-manylinux_2_17_aarch64.manylinux2014_aarch64.whl", hash = "sha256:8b37d5ec034e668b22cf0ce1074d6c21fd2a08b90d11b1b73139b750a8b0dd97", size = 338857 },
    { url = "https://files.pythonhosted.org/packages/a0/3d/54acbb3cdfcfea03d6a3535cff1e060a2de23e419a4e3955c9661171b8a8/yarl-1.16.0-cp311-cp311-manylinux_2_17_ppc64le.manylinux2014_ppc64le.whl", hash = "sha256:4f32c4cb7386b41936894685f6e093c8dfaf0960124d91fe0ec29fe439e201d0", size = 354005 },
    { url = "https://files.pythonhosted.org/packages/15/98/cd9fe3938422c88775c94578a6c145aca89ff8368ff64e6032213ac12403/yarl-1.16.0-cp311-cp311-manylinux_2_17_s390x.manylinux2014_s390x.whl", hash = "sha256:5b8e265a0545637492a7e12fd7038370d66c9375a61d88c5567d0e044ded9202", size = 351195 },
    { url = "https://files.pythonhosted.org/packages/e2/13/b6eff6ea1667aee948ecd6b1c8fb6473234f8e48f49af97be93251869c51/yarl-1.16.0-cp311-cp311-manylinux_2_17_x86_64.manylinux2014_x86_64.whl", hash = "sha256:789a3423f28a5fff46fbd04e339863c169ece97c827b44de16e1a7a42bc915d2", size = 342789 },
    { url = "https://files.pythonhosted.org/packages/fe/05/d98e65ea74a7e44bb033b2cf5bcc16edc1d5212bdc5ca7fbb5e380d89f8e/yarl-1.16.0-cp311-cp311-manylinux_2_5_i686.manylinux1_i686.manylinux_2_17_i686.manylinux2014_i686.whl", hash = "sha256:f1d1f45e3e8d37c804dca99ab3cf4ab3ed2e7a62cd82542924b14c0a4f46d243", size = 336478 },
    { url = "https://files.pythonhosted.org/packages/7d/47/43de2e94b75f36d84733a35c807d0e33aaf084e98f32e2cbc685102f4ba4/yarl-1.16.0-cp311-cp311-musllinux_1_2_aarch64.whl", hash = "sha256:621280719c4c5dad4c1391160a9b88925bb8b0ff6a7d5af3224643024871675f", size = 346008 },
    { url = "https://files.pythonhosted.org/packages/e2/de/9c2f900ec5e2f2e20329cfe7dcd9452e326d08cb5ecd098c2d4e9987b65c/yarl-1.16.0-cp311-cp311-musllinux_1_2_armv7l.whl", hash = "sha256:ed097b26f18a1f5ff05f661dc36528c5f6735ba4ce8c9645e83b064665131349", size = 343745 },
    { url = "https://files.pythonhosted.org/packages/56/cd/b014dce22e37b77caa37f998c6c47434fd78d01e7be07119629f369f5ee1/yarl-1.16.0-cp311-cp311-musllinux_1_2_i686.whl", hash = "sha256:2f1fe2b2e3ee418862f5ebc0c0083c97f6f6625781382f828f6d4e9b614eba9b", size = 349705 },
    { url = "https://files.pythonhosted.org/packages/07/17/bb191a26f7189423964e008ccb5146ce5258454ef3979f9d4c6860d282c7/yarl-1.16.0-cp311-cp311-musllinux_1_2_ppc64le.whl", hash = "sha256:87dd10bc0618991c66cee0cc65fa74a45f4ecb13bceec3c62d78ad2e42b27a16", size = 360767 },
    { url = "https://files.pythonhosted.org/packages/19/09/7d777369e151991b708a5b35280ea7444621d65af5f0545bcdce5d840867/yarl-1.16.0-cp311-cp311-musllinux_1_2_s390x.whl", hash = "sha256:4199db024b58a8abb2cfcedac7b1292c3ad421684571aeb622a02f242280e8d6", size = 364755 },
    { url = "https://files.pythonhosted.org/packages/00/32/7558997d1d2e53dab15f6db5db49fc6b412b63ede3cb8314e5dd7cff14fe/yarl-1.16.0-cp311-cp311-musllinux_1_2_x86_64.whl", hash = "sha256:99a9dcd4b71dd5f5f949737ab3f356cfc058c709b4f49833aeffedc2652dac56", size = 357087 },
    { url = "https://files.pythonhosted.org/packages/28/20/c49a95a30c57224e5fb0fc83235295684b041300ce508b71821cb042527d/yarl-1.16.0-cp311-cp311-win32.whl", hash = "sha256:a9394c65ae0ed95679717d391c862dece9afacd8fa311683fc8b4362ce8a410c", size = 83030 },
    { url = "https://files.pythonhosted.org/packages/75/e3/2a746721d6f32886d9bafccdb80174349f180ccae0a287f25ba4312a2618/yarl-1.16.0-cp311-cp311-win_amd64.whl", hash = "sha256:5b9101f528ae0f8f65ac9d64dda2bb0627de8a50344b2f582779f32fda747c1d", size = 89616 },
    { url = "https://files.pythonhosted.org/packages/3a/be/82f696c8ce0395c37f62b955202368086e5cc114d5bb9cb1b634cff5e01d/yarl-1.16.0-cp312-cp312-macosx_10_13_universal2.whl", hash = "sha256:4ffb7c129707dd76ced0a4a4128ff452cecf0b0e929f2668ea05a371d9e5c104", size = 141230 },
    { url = "https://files.pythonhosted.org/packages/38/60/45caaa748b53c4b0964f899879fcddc41faa4e0d12c6f0ae3311e8c151ff/yarl-1.16.0-cp312-cp312-macosx_10_13_x86_64.whl", hash = "sha256:1a5e9d8ce1185723419c487758d81ac2bde693711947032cce600ca7c9cda7d6", size = 93515 },
    { url = "https://files.pythonhosted.org/packages/54/bd/33aaca2f824dc1d630729e16e313797e8b24c8f7b6803307e5394274e443/yarl-1.16.0-cp312-cp312-macosx_11_0_arm64.whl", hash = "sha256:d743e3118b2640cef7768ea955378c3536482d95550222f908f392167fe62059", size = 91441 },
    { url = "https://files.pythonhosted.org/packages/af/fa/1ce8ca85489925aabdb8d2e7bbeaf74e7d3e6ac069779d6d6b9c7c62a8ed/yarl-1.16.0-cp312-cp312-manylinux_2_17_aarch64.manylinux2014_aarch64.whl", hash = "sha256:26768342f256e6e3c37533bf9433f5f15f3e59e3c14b2409098291b3efaceacb", size = 330871 },
    { url = "https://files.pythonhosted.org/packages/f1/2a/a8110a225e498b87315827f8b61d24de35f86041834cf8c9c5544380c46b/yarl-1.16.0-cp312-cp312-manylinux_2_17_ppc64le.manylinux2014_ppc64le.whl", hash = "sha256:d1b0796168b953bca6600c5f97f5ed407479889a36ad7d17183366260f29a6b9", size = 340641 },
    { url = "https://files.pythonhosted.org/packages/d0/64/20cd1cb1f60b3ff49e7d75c1a2083352e7c5939368aafa960712c9e53797/yarl-1.16.0-cp312-cp312-manylinux_2_17_s390x.manylinux2014_s390x.whl", hash = "sha256:858728086914f3a407aa7979cab743bbda1fe2bdf39ffcd991469a370dd7414d", size = 340245 },
    { url = "https://files.pythonhosted.org/packages/77/a8/7f38bbefb22eb925a68ad1d8193b05f51515614a6c0ebcadf26e9ae5e5ad/yarl-1.16.0-cp312-cp312-manylinux_2_17_x86_64.manylinux2014_x86_64.whl", hash = "sha256:5570e6d47bcb03215baf4c9ad7bf7c013e56285d9d35013541f9ac2b372593e7", size = 336054 },
    { url = "https://files.pythonhosted.org/packages/b4/a6/ac633ea3ea0c4eb1057e6800db1d077e77493b4b3449a4a97b2fbefadef4/yarl-1.16.0-cp312-cp312-manylinux_2_5_i686.manylinux1_i686.manylinux_2_17_i686.manylinux2014_i686.whl", hash = "sha256:66ea8311422a7ba1fc79b4c42c2baa10566469fe5a78500d4e7754d6e6db8724", size = 324405 },
    { url = "https://files.pythonhosted.org/packages/93/cd/4fc87ce9b0df7afb610ffb904f4aef25f59e0ad40a49da19a475facf98b7/yarl-1.16.0-cp312-cp312-musllinux_1_2_aarch64.whl", hash = "sha256:649bddcedee692ee8a9b7b6e38582cb4062dc4253de9711568e5620d8707c2a3", size = 342235 },
    { url = "https://files.pythonhosted.org/packages/9f/bc/38bae4b716da1206849d88e167d3d2c5695ae9b418a3915220947593e5ca/yarl-1.16.0-cp312-cp312-musllinux_1_2_armv7l.whl", hash = "sha256:3a91654adb7643cb21b46f04244c5a315a440dcad63213033826549fa2435f71", size = 340835 },
    { url = "https://files.pythonhosted.org/packages/dc/0f/b9efbc0075916a450cbad41299dff3bdd3393cb1d8378bb831c4a6a836e1/yarl-1.16.0-cp312-cp312-musllinux_1_2_i686.whl", hash = "sha256:b439cae82034ade094526a8f692b9a2b5ee936452de5e4c5f0f6c48df23f8604", size = 344323 },
    { url = "https://files.pythonhosted.org/packages/87/6d/dc483ea1574005f14ef4c5f5f726cf60327b07ac83bd417d98db23e5285f/yarl-1.16.0-cp312-cp312-musllinux_1_2_ppc64le.whl", hash = "sha256:571f781ae8ac463ce30bacebfaef2c6581543776d5970b2372fbe31d7bf31a07", size = 355112 },
    { url = "https://files.pythonhosted.org/packages/10/22/3b7c3728d26b3cc295c51160ae4e2612ab7d3f9df30beece44bf72861730/yarl-1.16.0-cp312-cp312-musllinux_1_2_s390x.whl", hash = "sha256:aa7943f04f36d6cafc0cf53ea89824ac2c37acbdb4b316a654176ab8ffd0f968", size = 361506 },
    { url = "https://files.pythonhosted.org/packages/ad/8d/b7b5d43cf22a020b564ddf7502d83df150d797e34f18f6bf5fe0f12cbd91/yarl-1.16.0-cp312-cp312-musllinux_1_2_x86_64.whl", hash = "sha256:1a5cf32539373ff39d97723e39a9283a7277cbf1224f7aef0c56c9598b6486c3", size = 355746 },
    { url = "https://files.pythonhosted.org/packages/d9/a6/a2098bf3f09d38eb540b2b192e180d9d41c2ff64b692783db2188f0a55e3/yarl-1.16.0-cp312-cp312-win32.whl", hash = "sha256:a5b6c09b9b4253d6a208b0f4a2f9206e511ec68dce9198e0fbec4f160137aa67", size = 82675 },
    { url = "https://files.pythonhosted.org/packages/ed/a6/0a54b382cfc336e772b72681d6816a99222dc2d21876e649474973b8d244/yarl-1.16.0-cp312-cp312-win_amd64.whl", hash = "sha256:1208ca14eed2fda324042adf8d6c0adf4a31522fa95e0929027cd487875f0240", size = 88986 },
    { url = "https://files.pythonhosted.org/packages/57/56/eef0a7050fcd11d70c536453f014d4b2dfd83fb934c9857fa1a912832405/yarl-1.16.0-cp313-cp313-macosx_10_13_universal2.whl", hash = "sha256:a5ace0177520bd4caa99295a9b6fb831d0e9a57d8e0501a22ffaa61b4c024283", size = 139373 },
    { url = "https://files.pythonhosted.org/packages/3f/b2/88eb9e98c5a4549606ebf673cba0d701f13ec855021b781f8e3fd7c04190/yarl-1.16.0-cp313-cp313-macosx_10_13_x86_64.whl", hash = "sha256:7118bdb5e3ed81acaa2095cba7ec02a0fe74b52a16ab9f9ac8e28e53ee299732", size = 92759 },
    { url = "https://files.pythonhosted.org/packages/95/1d/c3b794ef82a3b1894a9f8fc1012b073a85464b95c646ac217e8013137ea3/yarl-1.16.0-cp313-cp313-macosx_11_0_arm64.whl", hash = "sha256:38fec8a2a94c58bd47c9a50a45d321ab2285ad133adefbbadf3012c054b7e656", size = 90573 },
    { url = "https://files.pythonhosted.org/packages/7f/35/39a5dcbf7ef320607bcfd1c0498ce348181b97627c3901139b429d806cf1/yarl-1.16.0-cp313-cp313-manylinux_2_17_aarch64.manylinux2014_aarch64.whl", hash = "sha256:8791d66d81ee45866a7bb15a517b01a2bcf583a18ebf5d72a84e6064c417e64b", size = 332461 },
    { url = "https://files.pythonhosted.org/packages/36/29/2a468c8b44aa750d0f3416bc24d58464237b402388a8f03091a58537274a/yarl-1.16.0-cp313-cp313-manylinux_2_17_ppc64le.manylinux2014_ppc64le.whl", hash = "sha256:1cf936ba67bc6c734f3aa1c01391da74ab7fc046a9f8bbfa230b8393b90cf472", size = 343045 },
    { url = "https://files.pythonhosted.org/packages/91/6a/002300c86ed7ef3cd5ac890a0e17101aee06c64abe2e43f9dad85bc32c70/yarl-1.16.0-cp313-cp313-manylinux_2_17_s390x.manylinux2014_s390x.whl", hash = "sha256:d1aab176dd55b59f77a63b27cffaca67d29987d91a5b615cbead41331e6b7428", size = 344592 },
    { url = "https://files.pythonhosted.org/packages/ea/69/ca4228e0f560f0c5817e0ebd789690c78ab17e6a876b38a5d000889b2f63/yarl-1.16.0-cp313-cp313-manylinux_2_17_x86_64.manylinux2014_x86_64.whl", hash = "sha256:995d0759004c08abd5d1b81300a91d18c8577c6389300bed1c7c11675105a44d", size = 338127 },
    { url = "https://files.pythonhosted.org/packages/81/df/32eea6e5199f7298ec15cf708895f35a7d2899177ed556e6bdf6819462aa/yarl-1.16.0-cp313-cp313-manylinux_2_5_i686.manylinux1_i686.manylinux_2_17_i686.manylinux2014_i686.whl", hash = "sha256:1bc22e00edeb068f71967ab99081e9406cd56dbed864fc3a8259442999d71552", size = 326127 },
    { url = "https://files.pythonhosted.org/packages/9a/11/1a888df53acd3d1d4b8dc803e0c8ed4a4b6cabc2abe19e4de31aa6b86857/yarl-1.16.0-cp313-cp313-musllinux_1_2_aarch64.whl", hash = "sha256:35b4f7842154176523e0a63c9b871168c69b98065d05a4f637fce342a6a2693a", size = 345219 },
    { url = "https://files.pythonhosted.org/packages/34/88/44fd8b372c4c50c010e66c62bfb34e67d6bd217c973599e0ee03f74e74ec/yarl-1.16.0-cp313-cp313-musllinux_1_2_armv7l.whl", hash = "sha256:7ace71c4b7a0c41f317ae24be62bb61e9d80838d38acb20e70697c625e71f120", size = 339742 },
    { url = "https://files.pythonhosted.org/packages/ee/c8/eaa53bd40db61265cec09d3c432d8bcd8ab9fd3a9fc5b0afdd13ab27b4a8/yarl-1.16.0-cp313-cp313-musllinux_1_2_i686.whl", hash = "sha256:8f639e3f5795a6568aa4f7d2ac6057c757dcd187593679f035adbf12b892bb00", size = 344695 },
    { url = "https://files.pythonhosted.org/packages/1b/8f/b00aa91bd3bc8ef41781b13ac967c9c5c2e3ca0c516cffdd15ac035a1839/yarl-1.16.0-cp313-cp313-musllinux_1_2_ppc64le.whl", hash = "sha256:e8be3aff14f0120ad049121322b107f8a759be76a6a62138322d4c8a337a9e2c", size = 353617 },
    { url = "https://files.pythonhosted.org/packages/f1/88/8e86a28a840b8dc30c880fdde127f9610c56e55796a2cc969949b4a60fe7/yarl-1.16.0-cp313-cp313-musllinux_1_2_s390x.whl", hash = "sha256:122d8e7986043d0549e9eb23c7fd23be078be4b70c9eb42a20052b3d3149c6f2", size = 359911 },
    { url = "https://files.pythonhosted.org/packages/ee/61/9d59f7096fd72d5f68168ed8134773982ee48a8cb4009ecb34344e064999/yarl-1.16.0-cp313-cp313-musllinux_1_2_x86_64.whl", hash = "sha256:0fd9c227990f609c165f56b46107d0bc34553fe0387818c42c02f77974402c36", size = 358847 },
    { url = "https://files.pythonhosted.org/packages/f7/25/c323097b066a2b5a554f77e27a35bc067aebfcd3a001a0a3a6bc14190460/yarl-1.16.0-cp313-cp313-win32.whl", hash = "sha256:595ca5e943baed31d56b33b34736461a371c6ea0038d3baec399949dd628560b", size = 308302 },
    { url = "https://files.pythonhosted.org/packages/52/76/ca2c3de3511a127fc4124723e7ccc641aef5e0ec56c66d25dbd11f19ab84/yarl-1.16.0-cp313-cp313-win_amd64.whl", hash = "sha256:921b81b8d78f0e60242fb3db615ea3f368827a76af095d5a69f1c3366db3f596", size = 314035 },
    { url = "https://files.pythonhosted.org/packages/fb/f7/87a32867ddc1a9817018bfd6109ee57646a543acf0d272843d8393e575f9/yarl-1.16.0-py3-none-any.whl", hash = "sha256:e6980a558d8461230c457218bd6c92dfc1d10205548215c2c21d79dc8d0a96f3", size = 43746 },
]