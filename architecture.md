---
alwaysApply: true
---

# OUTLINE #
This is a complete guideline of how to write code in this repository.
Do not show too much freedom away from the instruction written in this markdown.
This architecture is named the "Mono-Lightweight" architecture.
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

```tsx
"use client";

import AuthPaper from "./widgets/AuthPaper";
import { Input } from "@/components/input";
import { useAuth } from "./hooks/use-auth";

export default function SignInForm() {
  const { loading, signIn, setEmail, setPassword, email, password } = useAuth();

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

## HOOKS ##

```tsx
"use client";

import { useDashboardStore, useUserStore, useSettingsStore } from "@/core/states";
import { handleSignIn } from "./actions/auth-features";
import { useState } from "react";
import { useRouter } from "next/navigation";
import { toast } from "sonner";

export function useAuth() {
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
      } = await handleSignIn(email, password);
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

```tsx
import { createClient } from "@/infrastructures/supabase/server";
import { Response } from "@/lib/types";
import { Tables } from "@/database.types";

export async function handleSignIn(email: string, password: string): Promise<Response<
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

  return { data: {user: userData as Tables<"user">, settings: settingsData as Tables<"settings">}, error: null };
}
```

--------------------

# LIB #

## UTILS ##

## TYPES ##

## STATES ##

## PROMPTS ##

--------------------

# DATA TYPE MANAGEMENT #
