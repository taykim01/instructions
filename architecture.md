---
alwaysApply: true
---

# OUTLINE #
This is a complete guideline of how to write code in this repository.
Do not show too much freedom away from the instruction written in this markdown.
This architecture is named the "Mono-Lightweight" architecture.
The goal of this architecture is to develop a website quickly while maintaining extreme code consistency.
This guideline provides coding guides that must be complied imperatively.

--------------------

# BACKGROUND INFORMATION #
This repository uses:
- Next.js
- Supabase
- Zustand
- Typescript
- Server Action (of Next.js)
- Shadcn (as base components)

This repository aims to construct a website using Next.js, easily scalable yet lean enough and customized for a single-developer environment.

--------------------

# ARCHITECTURE OVERVIEW #
The repository structure is as follows:
.
└── src/
    ├── app/
    │   ├── (pages)/
    │   │   └── sign-in.tsx
    │   ├── favicon.ico
    │   ├── globals.css
    │   └── layout.tsx
    ├── components
    ├── features/
    │   └── sign-in/
    │       ├── widget/
    │       │   ├── SignInButton
    │       │   └── SignInCard
    │       ├── action/
    │       │   └── sign-in.ts
    │       └── hook/
    │           └── use-sign-in.ts
    ├── infrastructures/
    │   ├── supabase/
    │   │   ├── client.ts
    │   │   ├── middleware.ts
    │   │   ├── names.ts
    │   │   └── server.ts
    │   └── openai/
    │       └── index.ts
    ├── lib/
    │   ├── utils
    │   ├── types
    │   ├── states/
    │   │   └── user-state.ts
    │   └── prompts
    └── database.types.ts
This the blueprint of the repository with only the sign in feature. There may be more features, pages, infrastructures added.

--------------------

# PAGES BLOCK #
The pages block refers to the files that construct the webpages in a Next.js project, within the src/app folder.
In the Mono-Lightweight architecture, the purpose of the page.tsx file is to show the abstract visual layout of the page.
In so doing, the complex state management and functions are hidden into the subordinate *widgets* (explained later).
Here is an example:
```tsx
export default function Page() {
  return (
    <AuthBackground>
      <SignInForm />
      <RouteContainer>
        <RouteToSignUp />
        <RouteToDemo />
      </RouteContainer>
    </AuthBackground>
  );
}
```
As you can see in this example, the page block is abstracted to its maximum, not showing how its widgets look or work, but only the basic layout of the page.
There is no need to add the `"use server"` directive, as it is set by default.

--------------------

# COMPONENT BLOCK #
The component block refers to the base UI pieces, such as base button, input, etc.
For this purpose, we use the Shadcn component set.
Generally, there is no need to modify this block, although adding new base components using the `npx shadcn@latest add` directive.
Because this block is merely for base components, handlers and functions must not be included.
These handlers and functions are to be added to the *widget* blocks.

--------------------

# FEATURE BLOCK #
The Mono-Lightweight architecture resembles the FSD architecture in that functions are grouped around each feature.
Each feature block contains:
- **widgets** for UI representation
- **actions** (which are Next.js's server actions) which handles communication with the database, server, etc.
- **hooks** (which are custom hooks) that preprocesses user's request, calls *actions*, and returns the results of the *action* to the client, including state management, alerts, toast, etc.

## WIDGETS ##
The widget block is a mixture of *component* block(or blocks) and hooks(as required).
The purpose of this block is to encapsulate a feature with a UI representation to simplify the *page* block.
A widget block may be as simple as a button with a sign in hook attached or may be complex as a form that takes all data and calls a hook handler.
The following is an example of a *widget* block.
Note: the `useAuth` hook provides the `email` and `password` variables as well as `setEmail` and `setPassword` handlers.
This is to simplify the *widget* block as much as possible and encapsulate all sub-functions related to a hook into that hook.
Check the **Pages Block** section and see how this <SignInForm /> is used.

```tsx
"use client";

import AuthPaper from "./widgets/AuthPaper";
import { Input } from "@/components/input";
import { useSignIn } from "./hooks/use-sign-in";

export default function SignInForm() {
  const { loading, signIn, setEmail, setPassword, email, password } = useSignIn();

  return (
    <AuthPaper
      title="Welcome Back"
      cta={{
        text: "Sign In",
        onClick: signIn,
        disabled: !email || !password,
        loading,
      }}
    >
      <div className="flex flex-col gap-5">
        <Input placeholder="Enter Email" label="Email" onChange={setEmail} value={email} onEnter={signIn} />
        <Input
          placeholder="Enter Password"
          label="Password"
          type="password"
          onChange={setPassword}
          value={password}
          onEnter={signIn}
        />
      </div>
    </AuthPaper>
  );
}
```

## ACTIONS ##
The *actions* block uses Next.js server actions.
An *action* does not:
1. route the user to another page
2. refresh the page
3. throw an error directly
Instead return a response or an error, encapsulated in a `Response` interface(see the `Response` interface below)
A REST style api call is necessary to use an exernal service or a service initiated by the infrastructure block happens in the *actions* block.
Every time an error is identified, throw a server console error.
See the example below.

```tsx
export interface Response<T> {
  data: T | null;
  error: string | null;
}
```

```tsx
"use server"

import { createClient } from "@/infrastructures/supabase/server";
import { Response } from "@/lib/types";
import { Tables } from "@/database.types";

export async function signInAction(email: string, password: string): Promise<Response<
  { user: Tables<"user">; settings: Tables<"settings"> } | null>> {
    
  const supabase = await createClient();
  const { data, error } = await supabase.auth.signInWithPassword({
    email,
    password,
  });
  if (error) {
    const isInvalidCredentials = error.message.includes("Invalid login credentials");
    console.error("Error signing in:", error.message);
    if (isInvalidCredentials) return { data: null, error: "Invalid email or password" };
    else return { data: null, error: error.message };
  }
  const userId = data?.user?.id;
  if (!userId) {
    console.error("Error signing in: User ID not found");
    return { data: null, error: "User ID not found" };
  }

  const { data: userData, error: userError } = await supabase.from("user").select("*").eq("id", userId).single();
  if (userError || !userData) {
    console.error("Error getting user data:", userError?.message || "User not found");
    return { data: null, error: userError?.message || "User not found" };
  }

  const { data: settingsData, error: settingsError } = await supabase.from("settings").select("*").eq("user_id", userId).single();
  if(settingsError || !settingsData) {
    const errMsg = settingsError?.message || "Settings not found";
    console.error("Error getting settings data:", errMsg);
    return { data: null, error: errMsg };
  }

  return { data: { user: userData as Tables<"user">, settings: settingsData as Tables<"settings">}, error: null };
}
```

## HOOKS ##
The *hook* block:
1. takes a user event(ex. click, input, etc.),
2. pre-processes the event,
3. calls an *action* block(ex. calling the `handleSignIn` below),
4. receives a response,
5. and modifies the client state as a response to the user(either a response to be displayed somewhere, a toast, an alert, Zustand state modification, etc.).
Note: the states necessary for handlers within the hook is encapsulated inside the hook and the function of the `useState` is exported to be used in a *widget*.
Try to include as much necessary logic in the hook as possible(ex. `isLoggedIn` variable) to make the code in the *widget* block concise.

```tsx
"use client";

import { useDashboardStore, useUserStore, useSettingsStore } from "@/core/states";
import { signInAction } from "./actions/auth-features";
import { useState } from "react";
import { useRouter } from "next/navigation";
import { toast } from "sonner";

export function useSignIn() {
  const { user, _setUser, _resetUser } = useUserStore();
  const { settings, _setSettings } = useSettingsStore();
  const { _setNotes, _setEdges } = useDashboardStore();
  const [loading, setLoading] = useState(false);
  const [error, setError] = useState<string | null>(null);
  const [email, setEmail] = useState("");
  const [password, setPassword] = useState("");
  const router = useRouter();

  const signIn = async () => {
    setLoading(true);
    try {
      const {
        data,
        error,
      } = await signInAction(email, password);
      const userData = data?.user;
      const settingsData = data?.settings;
      if (!userData || !settingsData) {
        throw new Error("User or settings data not found");
      }
      if (error) throw error;
      _setUser(userData!);
      _setSettings(settingsData!);
      router.push("/dashboard");
      toast("Signed in!");
    } catch (error) {
      setError(error as string);
      alert(error);
    } finally {
      setLoading(false);
    }
  };

  const isLoggedIn = user && Object.keys(user).length > 0;

  return {
    user,
    isLoggedIn,
    settings,
    loading,
    error,
    signIn,
    setEmail,
    email,
    password,
  };
}
```

--------------------

# INFRASTRUCTURE BLOCK #
The *infrastructure* block is where an external service is initiated with necessary keys.
This block is generally given from the developer.
**DO NOT** modify this block unless explicitly directed to.

```tsx
import OpenAI from "openai";

if (!process.env.OPENAI_API_KEY) {
  throw new Error("OPENAI_API_KEY is not set in environment variables");
}

export const openai = new OpenAI({
  apiKey: process.env.OPENAI_API_KEY,
});

export const EMBEDDING_MODEL = "text-embedding-3-large";
export const CHAT_MODELS = {
  gpt5: "gpt-5",
  gpt4omini: "gpt-4o-mini",
};

```

--------------------

# LIB #
The *lib* block is just a folder for all other necessary code, including **utils**, **types**, and **states**.

## STATES ##
The *states* part is used when a global state management is used.
Of note, this architecture heavily relies on global state management for the following reasons:
- each *widget* needs to be independent from each other, that is, not in the same inheritance tree, for code conciseness and readability issues;\
- a state modified in *widget* using a handler from a *hook* may need to be used in another *widget*.
Considering the characteristics of each state, use `persist` for optimization purposes.
For example, the `user` state, which holds information about a user's(received after logging in), is better off with persistance, as it does not change frequently.
Below is a state store initiator using Zustand with persistance.

```tsx
import { create } from "zustand";
import { persist } from "zustand/middleware";
import { UserState } from "../types";

const useUserStore = create<UserState>()(
  persist(
    (set) => ({
      user: {} as UserState["user"],
      _setUser: (user) => set({ user }),
      _resetUser: () => set({ user: {} as UserState["user"] }),
    }),
    {
      name: "user-store",
    },
  ),
);

export default useUserStore;

```

## PROMPTS ##
The use of the *prompts* section depends on whether an LLM services is used.
Prompts are separated into the *lib* block, as it is used in the *actions* block but holds irrelevant information for the execution of the function itself.
Use the `#{variable}` format; this is to extract a variable in the following method:
```tsx
const str = "Hello #{userName}, your id is #{user_id}.";
const vars = [...str.matchAll(re)].map(m => m[1]);
```

```tsx
export const queryClassificationPrompt = `
#ROLE#
You are a helpful assistant that classifies user's query into one of the following categories: [note, general, search].

#{variable}

'note' means that the user is asking a question about the user's notes, and the query needs to be searched in the user's notes.
'general' means that the query can be easily answered by the AI agent's response alone, without any context from user's notes and the internet.
'search' means that the user is looking for information that is not contained in their notes, and the query needs to be searched in the internet.

#RESPONSE FORMAT#
Response with only the category name, in the following JSON format:
{"classification": category}
`;
```

--------------------

# TABLE & TYPE MANAGEMENT #
This project uses Supabase as database.
Whenever a table is modified, use the following line ("$PROJECT_REF" being replaced with the actual project ID) to receive the updated database types.
```bash
npx supabase gen types typescript --project-id "$PROJECT_REF" --schema public > src/database.types.ts
```
