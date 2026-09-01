def route_feedback(feedback):
    text = feedback.lower().strip()

    if not text:
        return {"error": "Please enter some feedback."}

    # Agency keywords
    rules = {
        "PUB": [
            "drain", "flood", "flooding", "sewer",
            "water pipe", "water leak"
        ],
        "NEA": [
            "litter", "rubbish", "rat", "rats",
            "pest", "dirty", "pollution", "hawker"
        ],
        "LTA": [
            "road", "pothole", "traffic light",
            "bus stop", "traffic", "street sign"
        ],
        "NParks": [
            "park", "tree", "branch", "wildlife",
            "monkey", "greenery"
        ],
        "Town Council": [
            "corridor", "common area", "lift",
            "void deck", "estate light", "staircase"
        ],
        "HDB": [
            "inside my flat", "hdb policy",
            "ceiling crack", "flat renovation"
        ],
        "SLA": [
            "state land", "land boundary",
            "land ownership"
        ]
    }

    # Matters outside MSO's purview
    outside_keywords = [
        "income tax", "tax appeal", "immigration",
        "employment dispute", "court case",
        "neighbour owes me money", "private contract"
    ]

    if any(word in text for word in outside_keywords):
        return {
            "under_mso_purview": False,
            "primary_agency": "Outside MSO purview",
            "reason": "This does not appear to be a municipal-service matter.",
            "draft_reply": (
                "Thank you for your feedback. This matter does not appear "
                "to fall under the Municipal Services Office's purview. "
                "Please contact the relevant authority through its "
                "official channels."
            )
        }

    # Find the first matching agency
    for agency, keywords in rules.items():
        if any(word in text for word in keywords):
            return {
                "under_mso_purview": True,
                "primary_agency": agency,
                "reason": (
                    f"The feedback contains keywords associated "
                    f"with {agency}."
                ),
                "draft_reply": (
                    "Thank you for your feedback. Based on the information "
                    f"provided, the matter may be handled by {agency}. "
                    "Please provide the exact location if it was not "
                    "included. The recommendation should be reviewed "
                    "before the feedback is forwarded."
                )
            }

    # No keywords matched
    return {
        "under_mso_purview": "Unclear",
        "primary_agency": "Manual review required",
        "reason": "The issue did not match any of the routing keywords.",
        "draft_reply": (
            "Thank you for your feedback. More information is required "
            "to identify the responsible agency. Please provide the exact "
            "location and a clear description of the issue."
        )
    }


# Get feedback from the user
feedback = input("Enter public feedback: ")

# Analyse it
result = route_feedback(feedback)

# Display the output
print("\n" + "=" * 50)
print("MSO FEEDBACK ANALYSIS")
print("=" * 50)

if "error" in result:
    print(result["error"])
else:
    print("Under MSO purview:", result["under_mso_purview"])
    print("Primary agency:", result["primary_agency"])
    print("Reason:", result["reason"])
    print("\nDraft reply:")
    print(result["draft_reply"])

print("=" * 50)
